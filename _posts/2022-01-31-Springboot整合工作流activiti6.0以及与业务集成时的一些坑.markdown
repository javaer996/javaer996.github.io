---
layout: post
read_time: true
show_date: true
title:  Springboot整合工作流activiti6.0以及与业务集成时的一些坑
subtitle: 再苦，也别忘坚持^_^
date:   2022-01-31 12:33:20 +0800
description: Springboot整合工作流activiti6.0以及与业务集成时的一些坑.
categories: [教程]
tags: [springboot,activiti]
author: tengjiang
toc: yes
---


### 1、首先，要在springboot工程的pom文件中引入相关jar包

```html
<dependency>
      <groupId>org.activiti</groupId>
      <artifactId>activiti-spring</artifactId>
      <version>6.0.0</version>
</dependency>
```

### 2、添加配置文件，注入activiti的service

```java
@Configuration
public class ActivitiConfiguration {

    @Autowired
    private DataSource dataSource;
    @Autowired
    private PlatformTransactionManager platformTransactionManager;

	@Bean
	public SpringProcessEngineConfiguration springProcessEngineConfiguration(){
		SpringProcessEngineConfiguration spec = new SpringProcessEngineConfiguration();
        spec.setDataSource(dataSource);
        spec.setTransactionManager(platformTransactionManager);
		spec.setDatabaseSchemaUpdate("true");
		Resource[] resources = null;
        // 启动自动部署流程
		try {
			resources = new PathMatchingResourcePatternResolver().getResources("classpath*:bpmn/*.bpmn");
		} catch (IOException e) {
			e.printStackTrace();
		}
		spec.setDeploymentResources(resources);
		return spec;
	}
	
	@Bean
	public ProcessEngineFactoryBean processEngine(){
		ProcessEngineFactoryBean processEngineFactoryBean = new ProcessEngineFactoryBean();
		processEngineFactoryBean.setProcessEngineConfiguration(springProcessEngineConfiguration());
		return processEngineFactoryBean;
	}

	
	@Bean
	public RepositoryService repositoryService() throws Exception{
		return processEngine().getObject().getRepositoryService();
	}
	@Bean
	public RuntimeService runtimeService() throws Exception{
		return processEngine().getObject().getRuntimeService();
	}
	@Bean
	public TaskService taskService() throws Exception{
		return processEngine().getObject().getTaskService();
	}
	@Bean
	public HistoryService historyService() throws Exception{
		return processEngine().getObject().getHistoryService();
	}
	
}
```

### 3、与业务进行集成。

这里我使用activiti变量保存流程信息，在流程执行中使用流程监听器去修改业务表。为了简单，业务表和activiti的表在一个数据库内，这样就不会涉及分布式事务的问题了，直接使用spring的事务管理器就可以控制事务的提交，回滚了。

#### 3.1 画流程图

![10](https://s2.loli.net/2022/01/31/1NiDrgMtsGcnkaH.png)

#### 3.2设置流程信息

![20](https://s2.loli.net/2022/01/31/zpJNhlTRYX9Befo.png)

#### 3.3设置任务监听器

这里有不同的设置方式，我这里用的是expression，这样就不用实现activiti的JavaDelegate接口了，而且同一个类中可以实现很多方法，就像下面的apply方法就是自定义的一个方法。（Java Class和Delegate expression都需要实现接口才行）。

注意配置中的Event事件，这里可以选择监听器触发的时机，任务监听器有create，assignment，complete，all四种。 连线监听器有take，还有其他监听器请自行探索。

![30](https://s2.loli.net/2022/01/31/d6fn2zjJPIliZS3.png)

上面的apply方法中，我们可以看到有两个参数，其中execution是属于activiti内置的参数，java代码中只需要使用 DelegateExecution对象接收即可，从中我们可以获取流程相关的一些信息，而task是我自定义的一个参数，如果要想在监听器中获取该参数信息，我们需要在流程变量中设置名为task的参数。

变量分为全局的流程变量，局部的任务变量，和瞬时变量，因为这里的 task变量不需要存库，只是想在监听器中获取一下，所以这里使用的是瞬时变量。

> **注意点：**因为在流程启动的时候没有直接设置临时变量的地方，所以可以设置一个监听器，在启动流程实例的时候立即触发这个监听器， 然后在监听器中设置瞬时变量。

### 4、启动流程，并执行审核流程

#### 4.1 启动流程

```java
// 这里的myProcess就是我们上面流程中设置的Id，businessKey是业务key，可以用来和
// 业务进行绑定
runtimeService.startProcessInstanceByKey("myProcess", "businessKey");
```

#### 4.2执行审核流程

```java
// 通过业务key查询出对应的任务
Task task = taskService.createTaskQuery()
.processInstanceBusinessKey("businessKey").singleResult();
// 将相关信息放入工作流局部变量,和任务进行绑定
Map<String, Object> localVariable = new HashMap<>(2);
localVariable.put("user", "username");
localVariable.put("time" ,"2018-12-09");
taskService.setVariablesLocal(task.getId(), localVariable);

// 设置临时变量
Map<String, Object> variable = new HashMap<>(1);
variable.put("result", "pass");
// 完成任务并设置瞬时变量(这里还可以通过taskService直接设置瞬时变量)
taskService.complete(task.getId(),null,variable);
```

#### 4.3完成任务会触发我们的任务监听器（设置event为complete）

```java
    @Transactional(rollbackFor = Exception.class)
    public void apply(DelegateExecution execution,Task task){
        // 这里可以对我们的业务表进行操作，保证业务表和工作流中的状态是匹配的
    }
```

以上，权限是在业务表中进行控制的，所以的工作流执行的过程中，没有给任务指定任何处理人(工作流不指定任务处理人也可以一直往下执行)，在我的业务方法中，就相当于一直调用next方法，然后流程就会按照正常流程一直往下执行，直到结束。 如果是需要驳回等流程，那就需要单独处理判断了。如果需要设置处理人，只需要在任务的属性中进行配置即可。如下图：

![40](https://s2.loli.net/2022/01/31/wYzDoCj1f6uVPSl.png)

总结：使用工作流有两种方式

1、任务处理权限还是在业务表中进行控制，通过业务key查询任务并完成任务，工作流只是起一个流转以及保存流程历史信息的作用，对于业务来说，只要通过业务key找到流程任务，然后执行下一步即可，驳回等操作需要自己处理判断，然后完成任务后，自动触发作流的监听器去修改业务的具体数据。

2、任务处理权限由工作流控制，我们在工作流中指定任务的处理人(处理人可以动态指定)，然后通过处理人查询任务并完成任务，然后自动触发作流的监听器去修改业务的具体数据。

### **注意事项**

1、如果想要在监听器中获取到自定义任务变量，需要在任务监听器中进行设置任务变量，千万不要在任务之后的连线监听器中设置任务变量 (在一个事务中，没有流转到下一个任务，该任务属于还没有完成，所以在连线监听器中还能设置任务变量, 这样设置会导致任务变量保存到act_ru_variable表中，正常应该保存到act_hi_varinst表中）,不然设置的时候虽然不会报错，但是到最后结束流程的时候，你就会发现结束不了了，因为外键原因，act_ru_variable中存在任务变量没有删除，导出结束任务时，任务信息不能删除，然后报错！！！

2、瞬时变量的设置问题，启动流程的时候，没有直接设置瞬时变量的地方，所以可以在启动时通过设置一个监听器，触发监听器设置瞬时变量。但是，流程如果执行到了最后，后面没有用户任务节点了，这时候，千万不要再设置瞬时变量了，不然activiti会转换类型异常，它会将瞬时变量转为全局变量类型，这时就会报错，转换异常！！！