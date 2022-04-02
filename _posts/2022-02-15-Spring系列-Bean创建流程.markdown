---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring的Bean创建流程
subtitle: 疼痛挫折压力提醒着我们不要因为眼前的迷途而停滞不前。
date:   2022-02-15 09:32:20 +0800
description: Spring的Bean创建流程
categories: [Spring系列]
tags: [spring]
author: tengjiang
toc: yes
---


## Spring使用方式

我们使用Spring的时候，一般需要依赖于一个ApplicationContext实现类，我们分别从xml和注解两种方式介绍以下。

> 注意：这里不讨论使用tomcat等的web项目，我们只讨论普通java项目。无论哪种方式，每个实现类都有很多不同入参的构造方法，我们可以根据需要选择需要的构造方法。

### xml方式

示例类：`ClassPathXmlApplicationContext`

```java
// 第一种方式：这种方式我们不用调用refresh()方法，内部会自动调用
ApplicationContext cxac = new ClassPathXmlApplicationContext("spring.xml");  // 传入spring配置文件

// 第二种方式：这种方式我们需要手动调用refresh()方法
ApplicationContext cxac = new ClassPathXmlApplicationContext();
cxac.setConfigLocation("spring.xml");
cxac.refresh();
```

> 该方式在会在调用loadBeanDefinitions()的时候初始化XmlBeanDefinitionReader加载bean定义。

### 注解方式

实例类：`AnnotationConfigApplicationContext`

```java
// 同理，第一种方式，我们不用调用refresh()方法，内部会自动调用
ApplicationContext acac = new AnnotationConfigApplicationContext("com.demo");  // 传入需要扫描的包路径

// 第二种方式，这种方式需要我们手动调用refresh()方法
ApplicationContext acac = new AnnotationConfigApplicationContext();
acac.scan("com.demo");
acac.refresh();
```

> 该方式在创建该类的构造方法中就会初始化AnnotatedBeanDefinitionReader以及ClassPathBeanDefinitionScanner，用于读取bean定义。

## 启动入口refresh()

无论是使用哪一个实现类，入口只有一个，那就是AbstractApplicationContext中的refresh()方法。

refresh()方法的结构清晰明了，里面的每一个调用方法都见名知意，我们从中可以知道Spring启动时的整体执行逻辑是什么。

> 这里有一点需要说明，因为Spring中的各种实现类众多，继承关系也很复杂，所以对于不同的实现类同一个方法的实现逻辑可能是不同的。特别是对于使用xml方式和使用注解方式，它们都何时加载bean，如何加载bean有很大的区别，但是后续的实例化以及初始化bean就没什么区别了。
>
> 追踪源码的时候，可能我们想看一个实现逻辑的时候发现有好多类都实现了这个接口，这时我们怎样才知道我们到底应该看哪个类中的逻辑呢？其实我们只需要看我们最先是从哪个类开始的，然后看这个类的继承关系中到底继承的哪个类，我们就看哪个类的逻辑。

```java
// AbstractApplicationContext.java
public void refresh() throws BeansException, IllegalStateException {
  // 执行之前先加锁，避免并发执行
  synchronized(this.startupShutdownMonitor) {
    // 做一些刷新前的准备工作，记录一下启动时间，校验一下环境变量
    this.prepareRefresh();
    // 获取beanFacotory
    // 对于xml配置方式,如：ClassPathXmlApplicationContext会创建默认的DefaultListableBeanFacotory，还会加载bean定义
    // 对于注解方式，如：AnnotationConfigApplicationContext这里的beanFactory是直接获取的已经创建好的，此种方式在创建
    // AnnotationConfigApplicationContext时要么直接传入了自定义的beanFacotory，
    // 否则就已经自动创建了默认的DefaultListableBeanFacotory
    ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
    // 配置一下beanFacotory，主要是设置了一些处理器，需要忽略的接口等
    this.prepareBeanFactory(beanFactory);
    try {
      // 该方法是一个protected的方法，交由子类实现需要的设置, 
      // 例如AnnotationConfigServletWebServerApplicationContext在这里实现了包扫描逻辑
      this.postProcessBeanFactory(beanFactory);
      // 执行所有的beanFacotoryPostProcessors，这里我们既可以通过beanFacotoryRegister注册beanDefinition，
      // 也可以修改已经加载的beanDefinition的源信息
      this.invokeBeanFactoryPostProcessors(beanFactory);
      // 注册beanPostProcessor，包括我们自定义的，因为它会从beanFacotory中找beanPostProcessor，
      // 所以我们自定义的也可以加载进来，此时还没有实例化bean，
      // 所以我们自定义的beanPostProcessor可以在bean实例化和初始化的时候进行处理
      this.registerBeanPostProcessors(beanFactory);
      // 初始化国际化配置
      this.initMessageSource();
      // 初始化事件广播器
      this.initApplicationEventMulticaster();
      // 默认没有实现，子类中初始化了主题
      this.onRefresh();
      // 注册监听器，这里不仅会注册预先设置的监听器，还会查找beanFacotory中的监听器并注册，
      // 所以我们自定义的监听器在此时也被注册进去了
      this.registerListeners();
      // 这一步最重要，因为在这一步会实例化所有的单例bean
      this.finishBeanFactoryInitialization(beanFactory);
      // 启动完成，通过上下文完成事件
      this.finishRefresh();
    } catch (BeansException var9) {
    	if (this.logger.isWarnEnabled()) {
      this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
      }
			// 出现异常会销毁所有已创建的bean，然后停止
      this.destroyBeans();
      this.cancelRefresh(var9);
      throw var9;
    } finally {
      // 清除启动过程中的cache
    	this.resetCommonCaches();
    }
  }
}
```

## invokeBeanFactoryPostProcessors()

invokeBeanFactoryPostProcessors的逻辑也是在AbstractApplicationContext实现的。

上面说了，这里会执行所有的beanFacotoryPostProcessors的处理逻辑，里面可以注册新的beanDefinition(实现了BeanDefinitionRegistryPostProcessor接口)，也可以修改已有的beanDefinition(实现了BeanFactoryPostProcessors接口)。

```java
// AbstractApplicationContext.java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
  // 实际交由PostProcessorRegistrationDelegate类去执行
  PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, this.getBeanFactoryPostProcessors());
  ... // 省略无关代码
}
```
PostProcessorRegistrationDelegate中的步骤中可以大体分为几条线：

1. 后置处理器来源分为两种：
   - 参数传入
   - beanFactory容器中获取

2. 后置处理器类型分为两种：
   - 实现了BeanDefinitionRegistryPostProcessor接口的（可以注册beanDefinition和修改beanDefinition）
   - 实现了BeanFactoryPostProcessor接口的（可以修改beanDefinition）
3. 后置处理器顺序分为三种：
   - 实现了PriorityOrdered优先级接口的
   - 实现了Ordered排序接口的
   - 没有实现优先级或排序接口的

```java
// PostProcessorRegistrationDelegate.java
public static void invokeBeanFactoryPostProcessors(
  ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

  Set<String> processedBeans = new HashSet<>();
  // 前面我们说过了，这里主要是执行注册beanDefinition和修改beanDefinition的逻辑。
  // 首先处理的是注册beanDefinition，因为基本beanFactory实现类都会实现BeanDefinitionRegistry，所以这步基本都会处理
  if (beanFactory instanceof BeanDefinitionRegistry) {
    BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
    // 下面这两个集合存储的是全部后置处理器，包括 参数传入的 + 容器中获取的 
    // 用于临时存储BeanFactoryPostProcessor类型的后置处理器
    List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
    // 用户临时存储BeanDefinitionRegistryPostProcessor类型的后置处理器
    List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

    // 循环处理我们传递进来的所有后置处理器
    for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
      // 如果我们的处理器实现了BeanDefinitionRegistryPostProcessor接口，说明该处理器可以注册beanDefinition，那么
      // 调用该处理器的的注册方法，并将该处理器添加到registryProcessors集合中
      if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
        BeanDefinitionRegistryPostProcessor registryProcessor =
          (BeanDefinitionRegistryPostProcessor) postProcessor;
        // 调用注册方法
        registryProcessor.postProcessBeanDefinitionRegistry(registry);
        registryProcessors.add(registryProcessor);
      }
      else {
        // 如果没有实现BeanDefinitionRegistryPostProcessor接口，那么他就是BeanFactoryPostProcessor类型的处理器
        // 把当前的后置处理器加入到regularPostProcessors集合中
        regularPostProcessors.add(postProcessor);
      }
    }

    // 该集合用于存储当前需要执行处理的后置处理器，每一步都会将下面需要执行处理的后置处理器添加到该集合中
    List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

    // 上面执行的是传递进来的后置处理器，这里要从容器中获取所有符合条件的后置处理器
    String[] postProcessorNames =
      beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    for (String ppName : postProcessorNames) {
      // 优先处理实现了PriorityOrdered接口的后置处理器
      if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        // 通过getBean可以实例化以及初始化该后置处理器，将它添加到currentRegistryProcessors集合中，用于下面执行
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        // 将该后置处理器加入到processedBeans集合中去，避免后面重复处理
        processedBeans.add(ppName);
      }
    }
    // 对currentRegistryProcessors集合中BeanDefinitionRegistryPostProcessor进行排序
    sortPostProcessors(currentRegistryProcessors, beanFactory);
    // 将currentRegistryProcessors集合中的后置处理器添加到registryProcessors集合中
    registryProcessors.addAll(currentRegistryProcessors);
    /**
     * 在这里典型的BeanDefinitionRegistryPostProcessor就是ConfigurationClassPostProcessor
     * 用于进行bean定义的加载 比如我们的包扫描，@import等，我们所有加了注解的beanDefinition都是在
     * 这里被扫描出来的
     */
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    // 清空当前执行的后置处理器，因为这一批执行完了，下面还需要用该集合执行下一批
    currentRegistryProcessors.clear();

    postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    for (String ppName : postProcessorNames) {
      // 这里处理之前没有被处理过，且实现了Ordered接口的，下面的逻辑同上
      if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
      }
    }
    sortPostProcessors(currentRegistryProcessors, beanFactory);
    registryProcessors.addAll(currentRegistryProcessors);
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    currentRegistryProcessors.clear();

    /**
    *  定义一个重复处理的开关变量，为什么要定义这么一个变量呢？
    *  因为BeanDefinitionRegistryPostProcessor可以注册beanDefinition，如果该后置处理器中注册了其它的实现了
    *  BeanDefinitionRegistryPostProcessor接口的后置处理器，那么新注册的后置处理器也需要找出来，这样实现就可以
    *  找到所有的后置处理器了
    */
    boolean reiterate = true;
    while (reiterate) {
      reiterate = false;
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
        // 这里处理之前都没有被处理过的
        if (!processedBeans.contains(ppName)) {
          currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
          processedBeans.add(ppName);
          // 如果找到了未被处理的后置处理器，再次设置为true
          reiterate = true;
        }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();
    }

    // 上面都是执行的注册逻辑，这里执行的是修改逻辑，即BeanFactoryPostProcessors的postProcessBeanFactory()方法
    // 因为BeanDefinitionRegistryPostProcessor接口是继承了BeanFactoryPostProcessors接口的，所以无论哪个集合
    // 中的后置处理器都存在postProcessBeanFactory()方法
    invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
  }
  else { 
    // 如果beanFactory本身没有实现BeanDefinitionRegistry接口，说明该容器不支持注册beanDefinition，所以直接执行
    // postProcessBeanFactory()方法，忽略注册逻辑
    invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
  }

  // 获取容器中所有的BeanFactoryPostProcessor后置处理器，可能有人这里有疑问了，上面不是一遍遍的都获取过后置处理器了吗？
  // 注意这里获取的是BeanFactoryPostProcessor类型的，上面获取的都是BeanDefinitionRegistryPostProcessor类型的。
  // 不过这里需要注意的是，获取出来的后置处理器可能是包含上面已经处理了的处理器的，因为 
  // BeanDefinitionRegistryPostProcessor接口继承了BeanFactoryPostProcessors接口
  String[] postProcessorNames =
    beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

  // 这里也如上面一样，将后置处理器分为了实现PriorityOrdered接口的，实现Order接口的，没有实现接口的三种
  List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
  List<String> orderedPostProcessorNames = new ArrayList<>();
  List<String> nonOrderedPostProcessorNames = new ArrayList<>();
  for (String ppName : postProcessorNames) {
    // 如果processedBeans中包含的话，表示在上面处理BeanDefinitionRegistryPostProcessor类型处理器的时候处理过了
    if (processedBeans.contains(ppName)) {
      // skip - already processed in first phase above
    }
    else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
      priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
    }
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
      orderedPostProcessorNames.add(ppName);
    }
    else {
      nonOrderedPostProcessorNames.add(ppName);
    }
  }
  // 按照实现PriorityOrdered接口的，实现Order接口的，没有实现接口的顺序依次调用后置处理器的postProcessBeanFactory()方法
  sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

  List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
  for (String postProcessorName : orderedPostProcessorNames) {
    orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  }
  sortPostProcessors(orderedPostProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

  List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
  for (String postProcessorName : nonOrderedPostProcessorNames) {
    nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  }
  invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

  // Clear cached merged bean definitions since the post-processors might have
  // modified the original metadata, e.g. replacing placeholders in values...
  beanFactory.clearMetadataCache();
}
```

经过这一步，我们使用注解标注的Bean都已经生成了最终的BeanDefinition，接下来就可以那这些BeanDefinition去创建Bean了。

## finishBeanFactoryInitialization()

```java
// AbstractApplicationContext.java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
  if (beanFactory.containsBean("conversionService") && beanFactory.isTypeMatch("conversionService", ConversionService.class)) {
    // 设置conversionService，这也是一个很重要的功能，之后会写文章专门讲解
    beanFactory.setConversionService((ConversionService)beanFactory.getBean("conversionService", ConversionService.class));
  }
  ... // 忽略无关代码
  // 这里会立即实例化所有非懒加载的单例bean  
  beanFactory.preInstantiateSingletons();
}
```

## preInstantiateSingletons()

```java
// DefaultListableBeanFactory.java
public void preInstantiateSingletons() throws BeansException {
  // 获取我们容器中所有bean定义的名称
  List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

  for (String beanName : beanNames) {
    // 合并bean定义，这一步主要是做了什么呢？
    // 1. 将其它类型的beanDefinition转变为RootBeanDefinition，将有父子关系的beanDefinition合并
    RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		// 当该beanDefinition不是抽象的&&单例的&&不是懒加载的时候，会直接去实例化该bean
    if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
      // 判断是不是FactoryBean，工厂Bean也是一个Bean，注意不要和BeanFactory混了
      if (isFactoryBean(beanName)) {
        // 如果是工厂bean，给beanName前缀加上&符号，代表我们获取是FactoryBean，而不是通过FactoryBean.getObject()
        // 获取的实例对象
        // FactoryBean的作用就是可以让我们在实例化我们需要的真正的bean的实例可以定义一些个性化逻辑
        Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
        if (bean instanceof FactoryBean) {
          final FactoryBean<?> factory = (FactoryBean<?>) bean;
          boolean isEagerInit;
          if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
            isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                                        ((SmartFactoryBean<?>) factory)::isEagerInit,
                                                        getAccessControlContext());
          }
          else {
            isEagerInit = (factory instanceof SmartFactoryBean &&
                           ((SmartFactoryBean<?>) factory).isEagerInit());
          }
          // 判断是否需要立即通过FactoryBean创建真正的Bean实例
          if (isEagerInit) {
            getBean(beanName);
          }
        }
      }
      else {
        // 非工厂Bean，就是直接创建普通的bean
        getBean(beanName);
      }
    }
  }

  // 到这里所有的单实例的bean已经记载到单实例bean到缓存中了
  for (String beanName : beanNames) {
    Object singletonInstance = getSingleton(beanName);
    // 判断当前的bean是否实现了SmartInitializingSingleton接口，如果实现了的话，调用该bean的
    // afterSingletonsInstantiated()方法，注意执行这个方法的时候，该bean已经是一个完整的bean，
    // 所有的创建流程都已经走完了，该注入的信息也都已经注入了
    if (singletonInstance instanceof SmartInitializingSingleton) {
      final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
      if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
          smartSingleton.afterSingletonsInstantiated();
          return null;
        }, getAccessControlContext());
      }
      else {
        smartSingleton.afterSingletonsInstantiated();
      }
    }
  }
}
```

## getBean() -> doGetBean()

```java
// AbstractBeanFactory.java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

  // 在这里传入进来的name可能是别名, 也有可能是Factorybean的name,所以在这里需要转换一下名字
  final String beanName = transformedBeanName(name);
  Object bean;
  // 先尝试从缓存中拿单例bean，如果之前创建过，这里就能直接获取到，然后返回了
  Object sharedInstance = getSingleton(beanName);

  if (sharedInstance != null && args == null) {
    if (logger.isDebugEnabled()) {
      if (isSingletonCurrentlyInCreation(beanName)) {
        logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                     "' that is not fully initialized yet - a consequence of a circular reference");
      }
      else {
        logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
      }
    }
    /**
     * 如果 sharedInstance 是普通的单例 bean，下面的方法会直接返回。但如果
     * sharedInstance 是 FactoryBean 类型的，则需调用 getObject 工厂方法获取真正的
     * bean 实例。如果用户想获取 FactoryBean 本身，这里也不会做特别的处理，直接返回
     * 即可。毕竟 FactoryBean 的实现类本身也是一种 bean，只不过具有一点特殊的功能而已。
     */
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }
  else {
    // Spring只处理单例bean的循环依赖，prototype原型bean循环依赖Spring不会处理，直接报错
    if (isPrototypeCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(beanName);
    }
    // 判断AbstractBeanFacotry工厂是否有父工厂，一般情况下,只有Spring和SpringMvc整合的时才会有父子容器的概念,
    BeanFactory parentBeanFactory = getParentBeanFactory();
    // 若存在父工厂，并且当前的bean工厂不存在当前的bean定义，那么去父beanFacotry中获取该bean
    if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
      String nameToLookup = originalBeanName(name);
      if (parentBeanFactory instanceof AbstractBeanFactory) {
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
          nameToLookup, requiredType, args, typeCheckOnly);
      }
      else if (args != null) {
        return (T) parentBeanFactory.getBean(nameToLookup, args);
      }
      else {
        return parentBeanFactory.getBean(nameToLookup, requiredType);
      }
    }

    /**
     * 方法参数 typeCheckOnly ，是用来判断调用 getBean() 方法时，表示是否为仅仅进行类型检查获取Bean对象
     * 如果不是仅仅做类型检查，而是创建 Bean 对象，则需要调用 markBeanAsCreated()方法标记为已创建，因为下面
     * 就要开始真正创建了，避免出现重复创建
     */
    if (!typeCheckOnly) {
      markBeanAsCreated(beanName);
    }
    try {
      final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
      checkMergedBeanDefinition(mbd, beanName, args);

      // 处理dependsOn的依赖(这个不是我们所谓的循环依赖 而是bean创建前后的依赖),例如我们使用@DependsOn标注的类
      String[] dependsOn = mbd.getDependsOn();
      if (dependsOn != null) {
        for (String dep : dependsOn) {
          // 判断依赖bean和被依赖bean有没有循环依赖的关系，如果有，直接抛出异常
          // 因为bean创建的先后顺序依赖不能是循环依赖，否则我需要你先创建，你需要我先创建，那到底谁先创建？
          if (isDependent(beanName, dep)) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                            "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
          }
          // 注册依赖beanName之间的映射关系：依赖 beanName - > beanName 的集合
          registerDependentBean(dep, beanName);
          try {
            // 获取depentceOn的bean
            getBean(dep);
          }
          catch (NoSuchBeanDefinitionException ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                            "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
          }
        }
      }

      // 如果bean是单例的，那么实例化单例bean
      if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName, () -> {
          try {
            // 实际创建bean对象的方法
            return createBean(beanName, mbd, args);
          }
          catch (BeansException ex) {
            // 创建bean的过程中发生异常,需要销毁关于当前bean的所有信息
            destroySingleton(beanName);
            throw ex;
          }
        });
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
      }
      // 原型bean的创建
      else if (mbd.isPrototype()) {
        // It's a prototype -> create a new instance.
        Object prototypeInstance = null;
        try {
          beforePrototypeCreation(beanName);
          prototypeInstance = createBean(beanName, mbd, args);
        }
        finally {
          afterPrototypeCreation(beanName);
        }
        bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
      }
      else {
        // 指定特定scope的bean创建，这里以后会单独说
        String scopeName = mbd.getScope();
        final Scope scope = this.scopes.get(scopeName);
        if (scope == null) {
          throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
        }
        try {
          Object scopedInstance = scope.get(beanName, () -> {
            beforePrototypeCreation(beanName);
            try {
              return createBean(beanName, mbd, args);
            }
            finally {
              afterPrototypeCreation(beanName);
            }
          });
          bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
        }
        catch (IllegalStateException ex) {
          throw new BeanCreationException(beanName,
                                          "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                          "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                          ex);
        }
      }
    }
    catch (BeansException ex) {
      cleanupAfterBeanCreationFailure(beanName);
      throw ex;
    }
  }

  // 如果getBean的时候指定了想要获取的bean的类型，并且获取出来的bean不是指定的类型，那么这里会进行bean类型转换，转换是
  // 依靠类型转换器，即上面说的conversionService
  if (requiredType != null && !requiredType.isInstance(bean)) {
    try {
      T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
      if (convertedBean == null) {
        throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
      return convertedBean;
    }
    catch (TypeMismatchException ex) {
      if (logger.isDebugEnabled()) {
        logger.debug("Failed to convert bean '" + name + "' to required type '" +
                     ClassUtils.getQualifiedName(requiredType) + "'", ex);
      }
      throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
    }
  }
  return (T) bean;
}
```

## getSingleton()

这里有两个getSingletion()方法，一个是从缓存中获取bean的getSingleton()方法，一个是带有回调函数的getSingleton()方法。

### 从缓存中获取bean的getSingleton()

```java
// DefaultSingletonBeanRegistry.java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  // 从单例bean一级缓存中获取
  Object singletonObject = this.singletonObjects.get(beanName);
  // 如果没有获取到，并且要获取的该bean正在创建中，则从提前暴露bean的二级缓存中获取，注意此时获取的可能不是最终bean，
  // 如果最终bean对象与earlySingletonObjects中的对象不一致，会报错
  if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {
    synchronized(this.singletonObjects) {
      singletonObject = this.earlySingletonObjects.get(beanName);
      // 最后还没获取到，并且允许循环依赖的话，会在这里通过调用singletonFactories的方法提前暴露bean对象。
      // 这里是单级缓存，调用的其实就是BeanPostProcessor中的getEarlyBeanReference()
      if (singletonObject == null && allowEarlyReference) {
        ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);
        if (singletonFactory != null) {
          singletonObject = singletonFactory.getObject();
          this.earlySingletonObjects.put(beanName, singletonObject);
          this.singletonFactories.remove(beanName);
        }
      }
    }
  }
  return singletonObject;
}
```

### 回调函数的getSingleton()

```java
// DefaultSingletonBeanRegistry.java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
  Assert.notNull(beanName, "Bean name must not be null");
  synchronized (this.singletonObjects) {
    // 尝试从单例缓存池中获取对象
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null) {
      if (this.singletonsCurrentlyInDestruction) {
        throw new BeanCreationNotAllowedException(beanName,
                                                  "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                                                  "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
      }
      if (logger.isDebugEnabled()) {
        logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
      }
      // 标记当前的bean正在创建中
      // singletonsCurrentlyInCreation在这里会把beanName加入进来，若第二次循环依赖，构造器注入会抛出异常，
      // 因为构造器注入不允许循环依赖
      beforeSingletonCreation(beanName);
      boolean newSingleton = false;
      boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
      if (recordSuppressedExceptions) {
        this.suppressedExceptions = new LinkedHashSet<>();
      }
      try {
        // 这里其实是调用createBean()方法
        singletonObject = singletonFactory.getObject();
        newSingleton = true;
      }
      catch (IllegalStateException ex) {
        singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
          throw ex;
        }
      }
      catch (BeanCreationException ex) {
        if (recordSuppressedExceptions) {
          for (Exception suppressedException : this.suppressedExceptions) {
            ex.addRelatedCause(suppressedException);
          }
        }
        throw ex;
      }
      finally {
        if (recordSuppressedExceptions) {
          this.suppressedExceptions = null;
        }
        // 将该bean从标记为正在创建中集合里删除，因为已经创建完成了
        afterSingletonCreation(beanName);
      }
      if (newSingleton) {
        // 将创建好的bean加入单例bean一级缓存中
        addSingleton(beanName, singletonObject);
      }
    }
    return singletonObject;
  }
}
```

## createBean()

```java
// AbstractAutowireCapableBeanFactory.java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
  if (logger.isDebugEnabled()) {
    logger.debug("Creating instance of bean '" + beanName + "'");
  }
  RootBeanDefinition mbdToUse = mbd;
  // 确保此时的bean已经被解析了
  Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
  if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
    mbdToUse = new RootBeanDefinition(mbd);
    mbdToUse.setBeanClass(resolvedClass);
  }

  try {
    /**
      * 验证和准备覆盖方法
      * lookup-method 和 replace-method
      * 这两个配置存放在BeanDefinition中的methodOverrides
      * 我们知道在bean实例化的过程中如果检测到存在 methodOverrides ，
      * 则会动态地为当前bean生成代理并使用对应的拦截器为bean做增强处理。
      */
    mbdToUse.prepareMethodOverrides();
  }
  catch (BeanDefinitionValidationException ex) {
    throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
        beanName, "Validation of method overrides failed", ex);
  }

  try {
    /**
      * 在实例化bean之前先通过InstantiationAwareBeanPostProcessors的postProcessBeforeInstantiation()进行处理，
      * 如果返回了对象，则直接调用后置处理器的postProcessAfterInitialization()进行处理，然后将该对象当bean实例返回，
      * 不再执行之后的逻辑了。
      * 一般情况下在此处不会生成代理对象。不管是jdk代理还是cglib代理都不会在此处进行代理，因为我们的真实的对象没有生成,
      * 所以在这里不会生成代理对象。不过这一步是AOP和事务的关键，因为在这里解析我们的aop切面信息进行缓存。
      */
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
      return bean;
    }
  }
  catch (Throwable ex) {
    throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
        "BeanPostProcessor before instantiation of bean failed", ex);
  }

  try {
    // 该步骤是我们真正的创建我们的bean的实例对象的过程
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    if (logger.isDebugEnabled()) {
      logger.debug("Finished creating instance of bean '" + beanName + "'");
    }
    return beanInstance;
  }
  catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
    throw ex;
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
        mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
  }
}
```

## resolveBeforeInstantiation()

```java
// AbstractAutowireCapableBeanFactory.java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
  Object bean = null;
  if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
    // 判断容器中是否有InstantiationAwareBeanPostProcessors类型的后置处理器
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      // 推断当前bean的class对象
      Class<?> targetType = determineTargetType(beanName, mbd);
      if (targetType != null) {
				// 后置处理器的【第一次】调用，此处会解析出切面信息保存到缓存中
        bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
        // 如果InstantiationAwareBeanPostProcessors后置处理器的postProcessBeforeInstantiation返回不为null，
        // 直接调用所有后置处理器的postProcessAfterInitialization()方法
        if (bean != null) {
          // 后置处理器的【第二次】调用，该后置处理器若被调用的话，那么第一处的处理器肯定返回的不是null
          bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        }
      }
    }
    mbd.beforeInstantiationResolved = (bean != null);
  }
  return bean;
}
```

## doCreateBean()

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args) throws BeanCreationException {
  // BeanWrapper是对Bean的包装，其接口中所定义的功能很简单包括设置获取被包装的对象，获取被包装bean的属性描述器
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
  }
  if (instanceWrapper == null) {
    // 真正创建bean实例的地方，使用合适的实例化策略来创建新的实例：工厂方法、构造函数自动注入、简单初始化
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }
  final Object bean = instanceWrapper.getWrappedInstance();
  Class<?> beanType = instanceWrapper.getWrappedClass();
  if (beanType != NullBean.class) {
    mbd.resolvedTargetType = beanType;
  }

  synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
      try {
        // 后置处理器的【第三次】调用，这一步，会将@autowired和@value这类注解的元数据提取出来，为下面的注入做准备
        applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
      }
      catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                        "Post-processing of merged bean definition failed", ex);
      }
      mbd.postProcessed = true;
    }
  }

  // 是否允许提前暴露单例（是单例 && 允许循环依赖 && 该bean在创建中）
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                    isSingletonCurrentlyInCreation(beanName));
  // 第一次判断：如果允许提前暴露，则将bean放入singletonFacotries中, 在循环依赖的时候，可以通过
  // getEarlyBeanReference()提前自动代理，然后在其他bean中注入提前代理的对象
  if (earlySingletonExposure) {
    if (logger.isDebugEnabled()) {
      logger.debug("Eagerly caching bean '" + beanName +
                   "' to allow for resolving potential circular references");
    }
    // 后置处理器【第四次】调用
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }

  Object exposedObject = bean;
  try {
    // 执行属性注入
    populateBean(beanName, mbd, instanceWrapper);
    // 执行各种初始化方法
    exposedObject = initializeBean(beanName, exposedObject, mbd);
  }
  catch (Throwable ex) {
    if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
      throw (BeanCreationException) ex;
    }
    else {
      throw new BeanCreationException(
        mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
    }
  }
  /**
    * 第二次判断，主要是为了防止提前暴露的bean和最终实际的bean不一致，如果我们没有自定义或者引入会创建代理对象的
    * BeanPostProcessor的话，是不会有问题的，因为在spring的AbstractAutoProxyCreator中，如果已经通过
    * getEarlyBeanReference提前创建过代理对象了，那么在postProcessAfterInitialization中会直接跳过代理对象的创建, 
    * 然后将原bean对象返回，所以下面的exposedObject == bean是true, 又因为在postProcessAfterInitialization跳过了
    * 自动代理类的创建，但是在getEarlyBeanReference中代理了，所以要将earlySingletonReference赋给exposedObject，
    * 保证返回的bean和提前代理并注入的bean是一致的。
    * 所以如果我们如果想自定义BeanPostProcessor并要代理bean对象的话，需要按照AbstractAutoProxyCreator处理循环依赖
    * 的模式去实现，既要实现getEarlyBeanReference，还要实现postProcessAfterInitialization，如果调用过了
    * getEarlyBeanReference，提前暴露了bean对象，那么在postProcessAfterInitialization中要跳过
    * getEarlyBeanReference中的实现逻辑，而且不要改变bean对象
    */
  if (earlySingletonExposure) {
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
      if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
      }
      else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        String[] dependentBeans = getDependentBeans(beanName);
        Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
        for (String dependentBean : dependentBeans) {
          if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
            actualDependentBeans.add(dependentBean);
          }
        }
        if (!actualDependentBeans.isEmpty()) {
          throw new BeanCurrentlyInCreationException(beanName,
            "Bean with name '" + beanName + "' has been injected into other beans [" +
            StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
            "] in its raw version as part of a circular reference, but has eventually been " +
            "wrapped. This means that said other beans do not use the final version of the " +
            "bean. This is often the result of over-eager type matching - consider using " +
            "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
        }
      }
    }
  }

  // Register bean as disposable.
  try {
    // 后置处理器【第九次】调用，在销毁bean时，会调用DestructionAwareBeanPostProcessor类型后置处理器的
    // postProcessBeforeDestruction()方法
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
  }
  catch (BeanDefinitionValidationException ex) {
    throw new BeanCreationException(
      mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
  }

  return exposedObject;
}
```

扩展知识：

**spring解决循环依赖为什么需要三层缓存？**

> singletonObjects是存放已经处理完成的最终单例bean的。
>
> singletonFactories是存放用于提前暴露单例bean对象的对象工厂，如果存在循环依赖，才会调用对象工厂的方法提前暴露单例bean对象，否则不会调用。
>
> earlySingletonObjects是存放提前暴露还没有最终完成的单例对象的，此时它还不能算是单例bean对象。而且上面通过判断earlySingletonObjects中是否存在提前暴露的对象，以及判断提前暴露的对象和最终的bean对象是否是一致的来防止程序出错。

## populateBean()

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
  // 如果bw为null的话,则说明对象没有实例化
  if (bw == null) {
    // 进入if说明对象有属性，bw为空，不能为他设置属性，那就在下面就执行抛出异常
    if (mbd.hasPropertyValues()) {
      throw new BeanCreationException(
        mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
    }
    else {
      // Skip property population phase for null instance.
      return;
    }
  }

  /**
   * 在属性被填充前，给InstantiationAwareBeanPostProcessor类型的后置处理器一个修改bean状态的机会。
   * 官方的解释是：让用户可以自定义属性注入。比如用户实现一个InstantiationAwareBeanPostProcessor类型的后置处理器，
   * 并通过postProcessAfterInstantiation方法向bean的成员变量注入自定义的信息。
  */
  boolean continueWithPropertyPopulation = true;
  // 判断是否持有InstantiationAwareBeanPostProcessor
  if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
      if (bp instanceof InstantiationAwareBeanPostProcessor) {
        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
        // 后置处理器【第五次】调用，postProcessAfterInstantiation：如果应该继续在bean上面设置属性则返回true，
        // 否则返回false。如果返回了false，则不会继续执行下面的逻辑了
        if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
          continueWithPropertyPopulation = false;
          break;
        }
      }
    }
  }
  // 如果后续处理器发出停止填充命令，则终止后续操作
  if (!continueWithPropertyPopulation) {
    return;
  }
  //获取bean定义的属性
  PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
  /**
   * 判断我们的bean的属性注入模型，这里处理的xml配置的内容，注解配置的不是在这里处理
   * AUTOWIRE_BY_NAME 根据名称注入
   * AUTOWIRE_BY_TYPE 根据类型注入
   */
  if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
    // 把PropertyValues封装成为MutablePropertyValues
    MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
    // 根据bean的属性名称注入
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
      autowireByName(beanName, mbd, bw, newPvs);
    }
    // 根据bean的类型进行注入
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
      autowireByType(beanName, mbd, bw, newPvs);
    }
    // 把处理过的 属性覆盖原来的
    pvs = newPvs;
  }

  /**
   * 用于在Spring填充属性到bean对象前，对属性的值进行相应的处理，比如可以修改某些属性的值。
   * 这时注入到bean中的值就不是配置文件中的内容了，而是经过后置处理器修改后的内容。
   */
  boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
  // 判断是否需要检查依赖
  boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
  if (hasInstAwareBpps || needsDepCheck) {
    if (pvs == null) {
      pvs = mbd.getPropertyValues();
    }
    // 取出当前正在创建的beanWrapper依赖的对象
    PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
    if (hasInstAwareBpps) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
          InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
          // 后置处理器【第六次】调用，对依赖对象进行后置处理
          pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
          if (pvs == null) {
            return;
          }
        }
      }
    }
    //判断是否检查依赖
    if (needsDepCheck) {
      checkDependencies(beanName, mbd, filteredPds, pvs);
    }
  }

  /**
   * 其实，上面只是完成了所有注入属性的获取，将获取的属性封装在PropertyValues的实例对象 pvs 中，
   * 并没有应用到已经实例化的bean中。而applyPropertyValues() 方法，则是完成真正注入的。
   */
  if (pvs != null) {
    applyPropertyValues(beanName, mbd, bw, pvs);
  }
}
```

## initializeBean()

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd){
  if (System.getSecurityManager() != null) {
    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
      invokeAwareMethods(beanName, bean);
      return null;
    }, getAccessControlContext());
  }
  else {
    // 若我们的bean实现了XXXAware接口，在这里进行接口方法的回调，
    // 注意这里只是处理了BeanNameAware、BeanClassLoaderAware、BeanFactoryAware这三个接口的回调。
    // 与application有关的aware接口回调是在ApplicationContextAwareProcessor.postProcessBeforeInitialization()
    // 方法中处理的，包括：EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、	
    // ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware。
    // ImportAware是在ImportAwareBeanPostProcessor.postProcessBeforeInitialization()处理的。
    invokeAwareMethods(beanName, bean);
  }
  Object wrappedBean = bean;
  if (mbd == null || !mbd.isSynthetic()) {
    // 后置处理器【第七次】调用，调用我们的bean的后置处理器的postProcessorsBeforeInitialization方法  
    // @PostConstruct注解的方法就是在这一步被调用的
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  }

  try {
    // 调用初始化方法
    // InitializingBean接口，init-method方法
    invokeInitMethods(beanName, wrappedBean, mbd);
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
      (mbd != null ? mbd.getResourceDescription() : null),
      beanName, "Invocation of init method failed", ex);
  }
  if (mbd == null || !mbd.isSynthetic()) {
    // 后置处理器【第八次】调用，调用我们bean的后置处理器的PostProcessorsAfterInitialization方法
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  }
  return wrappedBean;
}
```
至此，Spring的bean创建流程就结束了。


