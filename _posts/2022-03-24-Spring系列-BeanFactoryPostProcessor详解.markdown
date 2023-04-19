---
layout: post
read_time: true
show_date: true
title:  Spring系列-BeanFactoryPostProcessor详解
subtitle: 
date:   2022-03-24 12:38:20 +0800
description: BeanFactoryPostProcessor BeanDefinitionRegistryPostProcessor
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---

## BeanFactoryPostProcessor是什么？

BeanFactoryPostProcessor是Bean工厂后置处理器，它是Spring的一个扩展点，可以让我们在Spring初始化Bean之前修改BeanDefinition或者注册BeanDefinition。接口定义如下：

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
	
  void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

## 调用时机

在之前的文章[Spring系列-Bean创建流程](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/02/15/Spring%E7%B3%BB%E5%88%97-Bean%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B.html)中我们已经讲过了，会在refresh()方法中的this.invokeBeanFactoryPostProcessors()方法中调用所有BeanFactoryPostProcessor的postProcessBeanFactory()方法。

## 自定义BeanFactoryPostProcessor

### 修改BeanDefinition

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    // 这里可以获取到已存在的BeanDefinition，并进行修改
    BeanDefinition bd = beanFactory.getBeanDefinition("test");
    bd.setPrimary(true);
    bd.setDescription("By BeanFactoryProcessor");
  }
}
```

### 注册BeanDefinition

注册这里需要注意的是，beanFactory实例必须是BeanDefinitionRegistry类型的才可以进行注册BeanDefinition。

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    BeanDefinition beanDefinition = new RootBeanDefinition();
    beanDefinition.setBeanClassName("com.demo.Test");
    beanDefinition.setScope(BeanDefinition.SCOPE_SINGLETON);
    beanDefinition.setAutowireCandidate(true);
    ((BeanDefinitionRegistry) beanDefinition).registerBeanDefinition("test", beanDefinition);
  }
}
```

## BeanDefinitionRegistryPostProcessor是什么？

BeanDefinitionRegistryPostProcessor继承了BeanFactoryPostProcessor接口，增加在专门用于注册BeanDefinition的生命周期回调接口，可以更方便的进行注册。它其实实际上和上面讲的使用BeanFactoryPostProcessor进行注册BeanDefinition没什么区别，其其作用的前提也是必须BeanFactory的实例是BeanDefinitionRegistry类型。接口定义如下：

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

  void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```

## 调用时机

同BeanFactoryPostProcessor，只不过postProcessBeanDefinitionRegistry()方法的调用在postProcessBeanFactory()之前。

## 自定义BeanDefinitionRegistryPostProcessor

```java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
  
  // 通过该方法注册BeanDefinition更加方便，不用再强转BeanDefinitionRegistry类型了
  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    BeanDefinition beanDefinition = new RootBeanDefinition();
    beanDefinition.setBeanClassName("com.demo.Test");
    beanDefinition.setScope(BeanDefinition.SCOPE_SINGLETON);
    beanDefinition.setAutowireCandidate(true);
    registry.registerBeanDefinition("test", beanDefinition);
  }

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
  }
}
```

## 用途

用于其它框架与Spring框架集成。

例如：mybatis与Spring集成时，Mapper接口的BeanDefinition的注册是通过MapperScannerConfigurer实现的，而MapperScannerConfigurer是实现了BeanDefinitionRegistryPostProcessor接口，在postProcessBeanDefinitionRegistry()方法中实现了Mapper接口的扫描，并进行了BeanDefinition的注册。