---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring事件机制详解
subtitle: 发光并非太阳的专利，你也可以发光。
date:   2022-04-06 18:02:20 +0800
description: Spring事件 ApplicationEvent @EventListener ApplicationListener applicationEventMulticaster
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---

Spring给我们提供了事件通知机制，下面我们就看一下Spring怎么使用事件通知以及事件通知的实现原理是什么。

## 自定义事件通知

自定义事件通知需要三步：

- 自定义事件
- 自定义事件监听器
- 发布事件

### 自定义事件

自定义事件类要继承org.springframework.context.ApplicationEvent抽象类，该抽象类实现了JDK中定义的java.util.EventObject接口。

```java
public class MyEvent extends ApplicationEvent {
    public MyEvent(Object source) {
        super(source);
    }
}
```

### 自定义事件监听器

自定义事件监听器有两种方式：

- 实现org.springframework.context.ApplicationListener接口
- 使用@EventListener注解

#### 实现ApplicationListener接口

```java
@Component
public class MyEventListener implements ApplicationListener<MyEvent> {
    @Override
    public void onApplicationEvent(MyEvent event) {
        System.out.println("MyEventListener ...");
    }
}
```

#### 使用@EventListener注解

使用注解有一个优势就是一个注解可以监听多个不同的事件，只需要在注解的value值中将需要监听的事件定义出来就可以，而且该注解支持SPEL表达式。但是使用注解也有一个缺点，就是该监听器只能监听到容器启动完成之后的事件通知，启动过程中的事件通知是不能监听到的。

```java
@Component
public class MyEventAnnotationListener{

    @EventListener(value = {MyEvent.class})
    public void onApplicationEvent(ApplicationEvent event) {
        System.out.println("@EventListener ...");
    }
}
```

### 发布事件

```java
public class Application {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext("com.demo.event");
        MyEvent myEvent = new MyEvent(applicationContext);
        // 因为AnnotationConfigApplicationContext实现了AnnotationConfigApplicationContext接口，所以可以直接发布事件
        applicationContext.publishEvent(myEvent);
    }
}
```

## 事件通知源码解析

### Spring容器刷新前处理

该处理是在refresh()方法中的prepareRefresh()方法中，不清楚refresh()方法的童鞋可以去[Spring系列-Bean的创建流程](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/02/15/Spring%E7%B3%BB%E5%88%97-Bean%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B.html)了解一下。

```java
// AbstractApplicationContext.java
protected void prepareRefresh() {
  // ...

  // 第一次earlyApplicationListeners肯定为null，此时初始化earlyApplicationListeners，并将applicationListeners中的监听器添加到earlyApplicationListeners
  // 此时applicationListeners中的监听器是怎么来的呢？我们可以通过applicationContext.addApplicationListener()方法手工添加监听器
  if (this.earlyApplicationListeners == null) {
    this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
  }
  else {
    // 如果重新刷新上下文，此时earlyApplicationListeners不为null了，这时将之前初始化的监听器清空，重置为第一次刷新的时候的earlyApplicationListeners中的监听器
    this.applicationListeners.clear();
    this.applicationListeners.addAll(this.earlyApplicationListeners);
  }

  // 将earlyApplicationEvents初始化，用于保存在applicationEventMulticaster没有创建之前发布的事件
  this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

### 初始化多播器

初始化多播器是在refresh()方法中的initApplicationEventMulticaster()中进行的初始化。

```java
// AbstractApplicationContext.java
protected void initApplicationEventMulticaster() {
  ConfigurableListableBeanFactory beanFactory = getBeanFactory();
  // 先判断beanFactory中存不存在名为applicationEventMulticaster的Bean，如果存在，将该bean赋值给applicationEventMulticaster
  if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
    this.applicationEventMulticaster =
      beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
    if (logger.isTraceEnabled()) {
      logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
    }
  }
  else {
    // 如果beanFactory中不存在，则创建一个SimpleApplicationEventMulticaster类型的实例赋值给applicationEventMulticaster，并将其注册为单例Bean
    // 这里是默认的，上面是给使用者一个使用自定义多播器的机会
    this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
    beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
    if (logger.isTraceEnabled()) {
      logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
                   "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
    }
  }
}
```

### 初始化监听器

初始化监听器同样是在refresh()方法中，不过是在initApplicationEventMulticaster()方法之后的registerListeners()方法中。

```java
// AbstractApplicationContext.java
protected void registerListeners() {
  // 注册静态的监听器
  // 什么是静态的监听器呢？就是我们通过applicationContext.addApplicationListener()方法手动添加的监听器
  for (ApplicationListener<?> listener : getApplicationListeners()) {
    // 可以看到这里是通过调用applicationEventMulticaster的addApplicationListener()方法进行注册的
    // (applicationContext中也存在addApplicationListener()方法，该方法不仅会将监听器保存到applicationContext的applicationListeners中，如果存在
    // applicationEventMulticaster，还会调用applicationEventMulticaster.addApplicationListener()方法)
    getApplicationEventMulticaster().addApplicationListener(listener);
  }

  // 找到所有实现了ApplicationListener接口的BeanName，这里之所以拿到BeanName而不是Bean实例，是因为这里还没有到单例Bean的实例化步骤，为了避免提前实例化实现了
  // ApplicationListener接口的Bean，这种实现了ApplicationListener接口的监听器，由ApplicationListenerDetector后置处理器处理
  // 这里为什么要获取BeanName呢？因为下面提前发布的事件可能需要通知未实例化的Bean，所以这里先找出来有哪些未实例化的ApplicationListener
  String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
  for (String listenerBeanName : listenerBeanNames) {
    // 先将实现了ApplicationListener接口的BeanName保存起来
    getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
  }

  // 看一下是否在applicationEventMulticaster没有创建之前就发布了事件，如果存在会将这些事件保存在earlyApplicationEvents集合中，在这里注册了监听器之后会将之前的事件
  // 进行通知
  Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
  // 将earlyApplicationEvents重新置为null，表示提前发布的事件处理了并且也不能往earlyApplicationEvents中添加事件了，因为applicationEventMulticaster已经实例化，可以
  // 直接进行事件发布了
  this.earlyApplicationEvents = null;
  if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
    for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
      // 注意这一步如果提前发布的事件涉及到了未实例化的Bean，这里会对Bean进行实例化，虽然此时还没有到单例Bean的实例化那一步，相当于提前实例化单例Bean了
      getApplicationEventMulticaster().multicastEvent(earlyEvent);
    }
  }
}
```

虽然可能由于earlyApplicationEvents事件的存在，已经提前实例化了一下ApplicationListener的Bean，正常的Bean的实例化是在finishBeanFactoryInitialization()触发的。

上面说了对于静态监听器直接就通过applicationEventMulticaster.addApplicationListener()进行添加了，那么对于实现了ApplicationListener接口的Bean呢？

是在ApplicationListenerDetector后置处理器中的postProcessAfterInitialization()方法中处理的。

```java
// ApplicationListenerDetector.java
public Object postProcessAfterInitialization(Object bean, String beanName) {
  // 如果实现了ApplicationListener接口则进行处理
  if (bean instanceof ApplicationListener) {
    Boolean flag = this.singletonNames.get(beanName);
    // 只有单例Bean才进行处理
    if (Boolean.TRUE.equals(flag)) {
      // 通过调用applicationContext.addApplicationListener()添加监听器
      this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
    }
    else if (Boolean.FALSE.equals(flag)) {
      this.singletonNames.remove(beanName);
    }
  }
  return bean;
}
```

```java
// AbstractApplicationContext.java
public void addApplicationListener(ApplicationListener<?> listener) {
  Assert.notNull(listener, "ApplicationListener must not be null");
  // 如果applicationEventMulticaster不为null，还要调用applicationEventMulticaster.addApplicationListener()方法添加监听器，因为事件通知实际上是拿的
  // applicationEventMulticaster中保存的监听器
  if (this.applicationEventMulticaster != null) {
    this.applicationEventMulticaster.addApplicationListener(listener);
  }
  // 将监听器添加到applicationListeners集合中
  this.applicationListeners.add(listener);
}
```

实际上在applicationEventMulticaster是由属性defaultRetriever中的applicationListeners属性持有监听器。

```java
// AbstractApplicationEventMulticaster.java
public void addApplicationListener(ApplicationListener<?> listener) {
  synchronized (this.defaultRetriever) {
    // 显式删除代理的目标，如果已经注册，为了避免同一个监听器的双重调用
    Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
    if (singletonTarget instanceof ApplicationListener) {
      this.defaultRetriever.applicationListeners.remove(singletonTarget);
    }
    // 通过defaultRetriever持有监听器，默认实现类是DefaultListenerRetriever
    this.defaultRetriever.applicationListeners.add(listener);
    // 清空缓存
    // 这里说明了只要添加一个新的事件监听器，就会将之前缓存的事件与对应事件监听器的缓存全部清空
    this.retrieverCache.clear();
  }
}
```

说完了直接实现接口的方式，那么对于@EventListener注解的方式是怎么处理的呢？

注解的方式在通过EventListenerMethodProcessor这个Bean工厂后置处理器处理的，该类实现了SmartInitializingSingleton接口，所以该接口定义的afterSingletonsInstantiated()方法在所有单例Bean都创建完成后才会调用，所以说注解方式的监听器监听不到再容器启动完成之前的事件。

在afterSingletonsInstantiated()调用了processBean()方法，在该方法中完成了注解监听器的处理。

```java
// EventListenerMethodProcessor.java
private void processBean(final String beanName, final Class<?> targetType) {
  if (!this.nonAnnotatedClasses.contains(targetType) &&
      AnnotationUtils.isCandidateClass(targetType, EventListener.class) &&
      !isSpringContainerClass(targetType)) {

    Map<Method, EventListener> annotatedMethods = null;
    try {
      // 查找该Bean中所有定义了@EventListener注解的方法
      annotatedMethods = MethodIntrospector.selectMethods(targetType,
                                                          (MethodIntrospector.MetadataLookup<EventListener>) method ->
                                                          AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));
    }
    catch (Throwable ex) {
      // An unresolvable type in a method signature, probably from a lazy bean - let's ignore it.
      if (logger.isDebugEnabled()) {
        logger.debug("Could not resolve methods for bean with name '" + beanName + "'", ex);
      }
    }

    if (CollectionUtils.isEmpty(annotatedMethods)) {
      this.nonAnnotatedClasses.add(targetType);
      if (logger.isTraceEnabled()) {
        logger.trace("No @EventListener annotations found on bean class: " + targetType.getName());
      }
    }
    else {
      ConfigurableApplicationContext context = this.applicationContext;
      Assert.state(context != null, "No ApplicationContext set");
      List<EventListenerFactory> factories = this.eventListenerFactories;
      Assert.state(factories != null, "EventListenerFactory List not initialized");
      // 通过EventListenerFactory处理注解监听器，默认是DefaultEventListenerFactory类
      for (Method method : annotatedMethods.keySet()) {
        for (EventListenerFactory factory : factories) {
          if (factory.supportsMethod(method)) {
            Method methodToUse = AopUtils.selectInvocableMethod(method, context.getType(beanName));
            // 通过EventListenerFactory将注解监听器封装为ApplicationListenerMethodAdapter类型
            ApplicationListener<?> applicationListener =
              factory.createApplicationListener(beanName, targetType, methodToUse);
            if (applicationListener instanceof ApplicationListenerMethodAdapter) {
              ((ApplicationListenerMethodAdapter) applicationListener).init(context, this.evaluator);
            }
            // 通过applicationContext.addApplicationListener()添加监听器
            context.addApplicationListener(applicationListener);
            break;
          }
        }
      }
      if (logger.isDebugEnabled()) {
        logger.debug(annotatedMethods.size() + " @EventListener methods processed on bean '" +
                     beanName + "': " + annotatedMethods);
      }
    }
  }
}
```

至此，监听器的创建与添加就完成了，下面就是事件的发布了。

### 发布事件

发布事件是通过事件多播器进行发布的，Spring默认的事件多播器是SimpleApplicationEventMulticaster类型。

```java
// SimpleApplicationEventMulticaster.java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
  ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
  // 获取任务执行器
  Executor executor = getTaskExecutor();
  // 获取指定事件类的事件监听器，并循环通知所有事件监听器
  for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
    // 如果有任务执行器，异步执行事件通知
    if (executor != null) {
      executor.execute(() -> invokeListener(listener, event));
    }
    else {
      // 没有任务执行器，同步执行时间通知
      invokeListener(listener, event);
    }
  }
}
```

事件通知之前要先拿到需要通知的所有事件监听器。

```java
// AbstractApplicationEventMulticaster.java
protected Collection<ApplicationListener<?>> getApplicationListeners(
  ApplicationEvent event, ResolvableType eventType) {
 
  Object source = event.getSource();
  Class<?> sourceType = (source != null ? source.getClass() : null);
  // 针对该事件的所有需要通知的事件监听器的缓存Key
  ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);
  // 缓存Key对应的所有监听器由CachedListenerRetriever对象保存
  CachedListenerRetriever newRetriever = null;

  // 先从缓存中获取，没有再执行真正的查找监听器逻辑
  CachedListenerRetriever existingRetriever = this.retrieverCache.get(cacheKey);
  if (existingRetriever == null) {
    if (this.beanClassLoader == null ||
        (ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
         (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
      // 创建缓存对象
      newRetriever = new CachedListenerRetriever();
      existingRetriever = this.retrieverCache.putIfAbsent(cacheKey, newRetriever);
      // 这里在此判断的目的是防止并发导致出现缓存覆盖现象，如果返回不为null，说明已经有线程创建缓存对象了
      if (existingRetriever != null) {
        newRetriever = null;  // no need to populate it in retrieveApplicationListeners
      }
    }
  }
  // 缓存对象存在，先从缓存对象中获取
  if (existingRetriever != null) {
    Collection<ApplicationListener<?>> result = existingRetriever.getApplicationListeners();
    if (result != null) {
      return result;
    }
  }
  // 执行真正的查找该事件需要通知的事件监听器的逻辑，这里将newRetriever传进去了，是为了将查找到的符合条件的监听器放到newRetriever保存起来
  return retrieveApplicationListeners(eventType, sourceType, newRetriever);
}
```

查找事件监听器的逻辑主要是通过前面说到的defaultRetriever中的两个属性处理，一个是applicationListeners，另一个是applicationListenerBeans，判断每一个监听器是否适用于该事件。

```java
// AbstractApplicationEventMulticaster.java
private Collection<ApplicationListener<?>> retrieveApplicationListeners(
  ResolvableType eventType, @Nullable Class<?> sourceType, @Nullable CachedListenerRetriever retriever) {

  List<ApplicationListener<?>> allListeners = new ArrayList<>();
  // 下面这两个集合是为了缓存使用
  Set<ApplicationListener<?>> filteredListeners = (retriever != null ? new LinkedHashSet<>() : null);
  Set<String> filteredListenerBeans = (retriever != null ? new LinkedHashSet<>() : null);

  Set<ApplicationListener<?>> listeners;
  Set<String> listenerBeans;
  synchronized (this.defaultRetriever) {
    listeners = new LinkedHashSet<>(this.defaultRetriever.applicationListeners);
    listenerBeans = new LinkedHashSet<>(this.defaultRetriever.applicationListenerBeans);
  }

  // 判断已经实例化的applicationListeners集合中的监听器是否适用于该事件
  for (ApplicationListener<?> listener : listeners) {
    if (supportsEvent(listener, eventType, sourceType)) {
      if (retriever != null) {
        filteredListeners.add(listener);
      }
      allListeners.add(listener);
    }
  }

  // 处理实现了ApplicationListener接口的BeanName，这里的BeanName对应的监听器可能部分已经实例化并添加到上面的applicationListeners中了
  if (!listenerBeans.isEmpty()) {
    ConfigurableBeanFactory beanFactory = getBeanFactory();
    for (String listenerBeanName : listenerBeans) {
      try {
        if (supportsEvent(beanFactory, listenerBeanName, eventType)) {
          ApplicationListener<?> listener =
            beanFactory.getBean(listenerBeanName, ApplicationListener.class);
          // 只有未实例化的才会处理，因为如果已经实例化，通过上面的listeners的处理，在allListeners中肯定已经存在了
          if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
            if (retriever != null) {
              if (beanFactory.isSingleton(listenerBeanName)) {
                filteredListeners.add(listener);
              }
              else {
                filteredListenerBeans.add(listenerBeanName);
              }
            }
            allListeners.add(listener);
          }
        }
        else {
          // Remove non-matching listeners that originally came from
          // ApplicationListenerDetector, possibly ruled out by additional
          // BeanDefinition metadata (e.g. factory method generics) above.
          Object listener = beanFactory.getSingleton(listenerBeanName);
          if (retriever != null) {
            filteredListeners.remove(listener);
          }
          allListeners.remove(listener);
        }
      }
      catch (NoSuchBeanDefinitionException ex) {
        // Singleton listener instance (without backing bean definition) disappeared -
        // probably in the middle of the destruction phase
      }
    }
  }
	// 对符合条件的监听器进行排序，说明监听器可能使用@Order注解进行排序
  AnnotationAwareOrderComparator.sort(allListeners);
  // 设置缓存
  if (retriever != null) {
    if (filteredListenerBeans.isEmpty()) {
      retriever.applicationListeners = new LinkedHashSet<>(allListeners);
      retriever.applicationListenerBeans = filteredListenerBeans;
    }
    else {
      retriever.applicationListeners = filteredListeners;
      retriever.applicationListenerBeans = filteredListenerBeans;
    }
  }
  return allListeners;
}
```

事件通知是通过invokeListener()方法中的doInvokeListener()方法调用的。

```java
// SimpleApplicationEventMulticaster.java
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
  try {
    // 调用监听器的onApplicationEvent()方法
    listener.onApplicationEvent(event);
  }
  catch (ClassCastException ex) {
    String msg = ex.getMessage();
    if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
      // Possibly a lambda-defined listener which we could not resolve the generic event type for
      // -> let's suppress the exception and just log a debug message.
      Log logger = LogFactory.getLog(getClass());
      if (logger.isTraceEnabled()) {
        logger.trace("Non-matching event type for listener: " + listener, ex);
      }
    }
    else {
      throw ex;
    }
  }
}
```

至此，Spring事件机制就说完了。