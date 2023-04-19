---
layout: post
read_time: true
show_date: true
title:  Spring系列-BeanPostProcessor详解
subtitle: 
date:   2022-03-24 12:38:20 +0800
description: BeanFactoryPostProcessor MergedBeanDefinitionPostProcessor InstantiationAwareBeanPostProcessor SmartInstantiationAwareBeanPostProcessor DestructionAwareBeanPostProcessor
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---

## BeanPostProcessor是什么？

同BeanFactoryPostProcessor一样，BeanPostProcessor也是Spring的一个扩展点，不同的是，BeanFactoryPostProcessor主要是用来处理BeanDefinition的，而BeanPostProcessor主要是用来处理Bean的(当前也有一种是用来处理BeanDefinition的)。BeanPostProcessor接口及其子接口实现类定义的方法覆盖了Spring创建Bean的各个生命周期，方便我们在需要的位置进行扩展。BeanPostProcessor定义如下：

```java
public interface BeanPostProcessor {

  // 该方法在Bean初始化之前调用
  // 一般注解标注的初始化方法会在这一步执行，例如：执行@PostConstruct注解标注的方法
  // 与ApplicationContext有关的Aware接口回调也会在这一步执行
  @Nullable
  default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    return bean;
  }

  // 该方法在Bean初始化之后调用
  // 这里是Bean创建完成之前最后的回调方法，一般动态代理就在这一步实现
  @Nullable
  default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    return bean;
  }
}
```

> 小贴士：实例化和初始化的区别？实例化->对象未创建，初始化->对象已创建。

下面的接口都是继承自BeanPostProcessor接口，提供更精细的生命周期回调方法。

## MergedBeanDefinitionPostProcessor

该接口与其它接口最大的不同是，其它接口都是处理Bean的，而该接口是处理BeanDefinition的。

```java
public interface MergedBeanDefinitionPostProcessor extends BeanPostProcessor {

  // 后处理指定bean的给定合并bean定义
  // 这里一般被实现为对于@Autowired、@Value、@PostConstruct、@PreDestroy等注解Metadata的提取
  void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);

  // 用于重置BeanDefinition相关信息
  // 这里一般被实现为对该Bean的一些缓存信息的清理
  default void resetBeanDefinition(String beanName) {
  }
}
```

## InstantiationAwareBeanPostProcessor

该接口主要定义了Bean的实例化前后以及属性处理回调接口。

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

  // 在Bean实例化之前调用
  // 值得注意的是，如果这里返回了一个对象实例，那么将在调用完所有后置处理器的postProcessAfterInitialization()方法后进行返回，将不再执行下面的实例化以及注入逻辑
  @Nullable
  default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
    return null;
  }
  
  // 在Bean实例化之后调用
  // 这里返回的是一个boolean，如果返回true, 表示需要继续执行属性处理以及属性注入逻辑，如果返回false，则表示跳过属性处理以及注入
  default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
    return true;
  }

  // 处理Bean属性注入信息
  // 例如：@Autowired注解的注入就是从AutowiredAnnotationBeanPostProcessor类的postProcessProperties()方法处理的
  @Nullable
  default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
      throws BeansException {

    return null;
  }

  // 已过期，从Spring5.1开始由上面的postProcessProperties()取代
  @Deprecated
  @Nullable
  default PropertyValues postProcessPropertyValues(
      PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

    return pvs;
  }
}
```

## SmartInstantiationAwareBeanPostProcessor

该接口主要定义提供推断Bean类型和构造方法的能力，最重要的是可以解决单例Bean在非构造方法注入时的循环依赖的问题。

```java
public interface SmartInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessor {

  // 用于推断Bean的类型
  @Nullable
  default Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException {
    return null;
  }

  // 用于推断BeanClass的所有构造方法
  @Nullable
  default Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName)
    throws BeansException {
    return null;
  }

  // 用于提前暴露Bean实例，用于解决循环依赖
  default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    return bean;
  }
}
```

## DestructionAwareBeanPostProcessor

该接口定义可以感知Bean的销毁，在Bean销毁之前做一些处理。

```java
public interface DestructionAwareBeanPostProcessor extends BeanPostProcessor {
  
  // 在Bean销毁之前调用, 例如调用@PreDestroy标注的方法
  void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;
  
  // 判断是否需要执行销毁方法
  default boolean requiresDestruction(Object bean) {
    return true;
  }
}
```