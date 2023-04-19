---
layout: post
read_time: true
show_date: true
title:  Spring系列-自定义Spring Scope
subtitle: 
date:   2022-04-01 10:42:20 +0800
description: Spring Scope @Scope
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---


上一节我们在[Spring系列-Spring Scope详解](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/03/30/Spring%E7%B3%BB%E5%88%97-Spring-Scope%E8%AF%A6%E8%A7%A3.html)讲了Spring Scope怎么使用以及Scope代理是怎么实现的，这一节我们将如何自定义Spring Scope。

自定义Spring Scope只需要三步：

- 定义Scope实现类
- 注册Scope实现类
- 使用自定义Scope

## 定义Scope实现类

定义Scope实现类需要实现org.springframework.beans.factory.config.Scope接口。例如我们定义一个Socpe，对于该Scope标识的Bean只有一个小时的有效期，超过一小时再次获取就要新建一个Bean。

```java
public class OneHourScope implements Scope {

  private static final long ONE_HOUR = 1 * 60 * 60 * 1000;
  // 保存每一个beanName最后一次创建时间
  private Map<String, Long> times = new ConcurrentHashMap<>();
  // 保存每一个beanName对应的锁对象
  private Map<String, Object> locks = new ConcurrentHashMap<>();
  // 缓存每一个beanName对应的Bean对象
  private Map<String, Object> cache = new ConcurrentHashMap<>();

  // 该方法用户获取对应的Bean对象
  @Override
  public Object get(String name, ObjectFactory<?> objectFactory) {
    long now = System.currentTimeMillis();
    long createTime = times.getOrDefault(name, 0L);
    if (createTime == 0) {
      Object lock = locks.get(name);
      if (lock == null) {
        lock = locks.putIfAbsent(name, new Object());
      }
      synchronized (lock) {
        Object object = cache.get(name);
        if (object == null) {
          // 第一次获取该Bean走创建逻辑，然后放入缓存
          object = objectFactory.getObject();
          cache.put(name, object);
          times.put(name, now);
        }
        return object;
      }
    }
    if (now - createTime < ONE_HOUR) {
      return cache.get(name);
    }
    Object lock = locks.get(name);
    synchronized (lock) {
      createTime = times.get(name);
      if (now - createTime > ONE_HOUR) {
        // 如果该Bean已经创建超过一个小时，则重新创建
        Object object = objectFactory.getObject();
        cache.put(name, object);
        times.put(name, now);
        return object;
      }
      return cache.get(name);
    }
  }

  // 该方法用于在销毁bean时进行一些处理
  @Override
  public Object remove(String name) {
    times.remove(name);
    locks.remove(name);
    return cache.remove(name);
  }

  // 该方法用于注册销毁Bean时的回调方法
  @Override
  public void registerDestructionCallback(String name, Runnable callback) {
  }

  // 该方法可用于针对该Scope进行域内变量解析
  @Override
  public Object resolveContextualObject(String key) {
    return null;
  }

  // 获取一个标识ID
  @Override
  public String getConversationId() {
    return null;
  }
}
```

## 注册Scope

通过CustomScopeConfigurer进行注册Scope，因为该类实现了BeanFactoryPostProcessor，会在postProcessBeanFactory()方法中将我们添加的Scope通过beanFactory.registerScope()进行注册。

```java
@Configuration
public class ScopeConfig {

  @Bean
  public CustomScopeConfigurer customScopeConfigurer() {
    CustomScopeConfigurer customScopeConfigurer = new CustomScopeConfigurer();
    customScopeConfigurer.addScope("oneHour", new OneHourScope());
    return customScopeConfigurer;
  }

}
```

## 使用Scope

使用@Scope注解标识该类，并设置Scope名为oneHour,并使用Scope代理。

```java
@Scope(value = "oneHour", proxyMode = ScopedProxyMode.TARGET_CLASS)
@Component
public class ScopeTest {

  public void test() {
    System.out.println(this);
  }

}
```

将标识了@Scope注解的类注入到其它类。

```java
@Component
public class Demo {
  // 这里注入的是一个代理对象，如果不断调用scopeTest.test()方法，每隔一小时就会换一个新的ScopeTest对象
  @Autowired
  private ScopeTest scopeTest;

}
```

Scope的基本使用很简单，其中Scope接口中有一个方法我们没说，就是resolveContextualObject(), 该方法可以用于解析Scope域内的变量，以后再说吧，O(∩_∩)O哈哈~。




