---
layout: post
read_time: true
show_date: true
title:  Spring系列-ObjectFactory与OjbectProvider
subtitle: 
date:   2022-04-07 15:13:20 +0800
description: ObjectFactory ObjectProvider
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---

在之前讲解[Spring系列-依赖注入实现原理-注解配置](https://www.tengjiang.site/spring/2022/03/10/Spring%E7%B3%BB%E5%88%97-%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86-%E6%B3%A8%E8%A7%A3%E9%85%8D%E7%BD%AE.html)的时候，在解析依赖**resolveDependency()**方法中已经碰到过了ObjectFactory和ObjectProvider,在那里我们说了它们可以解决懒加载以及依赖注入为null的问题，这一节我们就来详细讲讲这两个类。

## 接口定义

### ObjectFactory

我们先来看一下ObjectFactory的接口定义：

```java
@FunctionalInterface
public interface ObjectFactory<T> {
  T getObject() throws BeansException;
}
```

通过类名我们可以知道这是一个对象工厂，那肯定是用来生产对象的。而且我们发现该接口是一个函数式接口，只有一个方法定义getObject()用于返回一个对象实例。

看到getObect()方法不知道大家有没有感到很熟悉？没错，在FactoryBean中也有一个getObject()方法，只不过FactoryBean虽然也可以通过getObject()方法获取一个对象，但是FactoryBean本质上一个Bean，它在Spring中非常特殊，Spring对它进行了单独的处理。如果我们通过BeanName获取Bean时，发现该Bean是一个FactoryBean，那么Spring会自动调用FactoryBean的getObject()方法，然后将该方法返回的对象实例返回，如果我们就想获取FactoryBean实例，那么我们需要在BeanName的前面加上&符号，详细流程我们已经在[Spring系列-Spring的Bean创建流程](https://www.tengjiang.site/spring/2022/02/15/Spring%E7%B3%BB%E5%88%97-Bean%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B.html)介绍过了。

### ObjectProvier

我们再来看一下ObjectProvider的接口定义：

```java
public interface ObjectProvider<T> extends ObjectFactory<T>, Iterable<T> {

  // 返回用指定参数创建的bean, 如果容器中不存在, 则抛出异常
  T getObject(Object... args) throws BeansException;

  // 如果指定类型的在Bean中存在, 返回bean实例, 否则返回null
  @Nullable
  T getIfAvailable() throws BeansException;

  // 如果返回对象不存在，则用传入的Supplier获取一个Bean并返回，否则直接返回存在的对象
  default T getIfAvailable(Supplier<T> defaultSupplier) throws BeansException {
    T dependency = getIfAvailable();
    return (dependency != null ? dependency : defaultSupplier.get());
  }

  // 消费对象的一个实例，如果存在通过Consumer回调消耗目标对象, 如果不存在则直接返回
  default void ifAvailable(Consumer<T> dependencyConsumer) throws BeansException {
    T dependency = getIfAvailable();
    if (dependency != null) {
      dependencyConsumer.accept(dependency);
    }
  }

  // 如果不可用或不唯一（没有指定primary）则返回null。否则，返回对象
  @Nullable
  T getIfUnique() throws BeansException;

  // 如果不存在唯一对象，则调用Supplier的回调函数
  default T getIfUnique(Supplier<T> defaultSupplier) throws BeansException {
    T dependency = getIfUnique();
    return (dependency != null ? dependency : defaultSupplier.get());
  }

  // 如果存在唯一对象，则消耗掉该对象
  default void ifUnique(Consumer<T> dependencyConsumer) throws BeansException {
    T dependency = getIfUnique();
    if (dependency != null) {
      dependencyConsumer.accept(dependency);
    }
  }

  // 返回符合条件的对象的Iterator，没有特殊顺序保证（一般为注册顺序）
  @Override
  default Iterator<T> iterator() {
    return stream().iterator();
  }

  // 返回符合条件对象的连续的Stream，没有特殊顺序保证（一般为注册顺序）
  default Stream<T> stream() {
    throw new UnsupportedOperationException("Multi element access not supported");
  }

  // 返回符合条件对象的连续的Stream。在标注Spring应用上下文中采用@Order注解或实现Order接口的顺序
  default Stream<T> orderedStream() {
    throw new UnsupportedOperationException("Ordered element access not supported");
  }
}
```

同样通过类名我们可以知道这也是一个用于提供对象实例的工厂类，与ObjectFactory不同的是，它继承了ObjectFactory接口，并实现了Iterable接口，说明它扩展了OjbectFactory提供了更加丰富的功能。

## Spring应用场景

### 将ObjectFactory作为回调函数使用

在AbstractBeanFactory中的doGetBean()方法中进行使用。

```java
protected <T> T doGetBean(
  String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
  throws BeansException {
  // ...

  if (mbd.isSingleton()) {
    // 这里的getSingleton()方法的第二个参数就是ObjectFactory类型
    sharedInstance = getSingleton(beanName, () -> {
      try {
        return createBean(beanName, mbd, args);
      }
      catch (BeansException ex) {
        destroySingleton(beanName);
        throw ex;
      }
    });
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
  }else if (mbd.isPrototype()) {
    // ...
  }else {
    String scopeName = mbd.getScope();
    Scope scope = this.scopes.get(scopeName);
    try {
      // 这里的scope.get()方法的第二个参数也是ObjectFactory类型
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
  // ...
  return (T) bean;
}
```

### 使用ObjectFactory类型进行依赖注入

#### 注入ObjectFactory类型

> 这样如果实际要注入的Bean不存在，在使用时才会报错。

依赖注入

```java
@Component
public class Home {
  // 注入的类型为ObjectFactory, 通过user.getObject()获取真正的UserBean对象，如果不存在会报错
  @Autowired
  private ObjectFactory<User> user;
}
```

解析依赖注入

```java
// DefaultListableBeanFactory.java
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
                                @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

  descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
  if (Optional.class == descriptor.getDependencyType()) {
    return createOptionalDependency(descriptor, requestingBeanName);
  }
  // 如果是ObjectFactory类型或者ObjectProvider类型，创建一个DependencyObjectProvider对象返回，该类实现了ObjectProvider接口，实现了getObject()等方法
  else if (ObjectFactory.class == descriptor.getDependencyType() ||
           ObjectProvider.class == descriptor.getDependencyType()) {
    return new DependencyObjectProvider(descriptor, requestingBeanName);
  }
  else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
    return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
  }
  else {
    Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
      descriptor, requestingBeanName);
    if (result == null) {
      result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
    }
    return result;
  }
}
```

#### 注册解析可解析依赖

> 以依赖注入ServletRequest为例。

注册可解析依赖

```java
// 这里表明当需要注入ServletRequest类型的时候，实际上是使用RequestObjectFactory，而RequestObjectFactory实现了ObjectFactory接口
// 该方法会将它们之间的关联关系保存到resolvableDependencies集合中
beanFactory.registerResolvableDependency(ServletRequest.class, new RequestObjectFactory());
```

处理可解析依赖

处理逻辑是在resolveDependency() -> doResolveDependency() -> findAutowireCandidates()方法中。

```java
// DefaultListableBeanFactory.java
protected Map<String, Object> findAutowireCandidates(
  @Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {

  String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
    this, requiredType, true, descriptor.isEager());
  Map<String, Object> result = new LinkedHashMap<>(candidateNames.length);
  // 循环判断要注入的类型是否存在于可解析依赖的集合中
  for (Map.Entry<Class<?>, Object> classObjectEntry : this.resolvableDependencies.entrySet()) {
    Class<?> autowiringType = classObjectEntry.getKey();
    // 如果需要注入的类型是注册的类型或是其子类则通过
    if (autowiringType.isAssignableFrom(requiredType)) {
      Object autowiringValue = classObjectEntry.getValue();
      // 解析注入的对象，这里主要是为了生成代理对象，通过代理对象实现ObjectFactory对象的getObject()方法调用，最终调用到实际的Bean方法中
      autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
      if (requiredType.isInstance(autowiringValue)) {
        result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
        break;
      }
    }
  }
  // ...
  return result;
}
```

```java
// AutowireUtils.java
public static Object resolveAutowiringValue(Object autowiringValue, Class<?> requiredType) {
  // 如果可解析依赖的value对象是ObjectFactory类型并且需要注入的类型和该value对象的实际类型不一致，则需要创建代理
  if (autowiringValue instanceof ObjectFactory && !requiredType.isInstance(autowiringValue)) {
    ObjectFactory<?> factory = (ObjectFactory<?>) autowiringValue;
    if (autowiringValue instanceof Serializable && requiredType.isInterface()) {
      // 创建代理对象，handler为ObjectFactoryDelegatingInvocationHandler，其持有ObjectFactory实现类对象
      autowiringValue = Proxy.newProxyInstance(requiredType.getClassLoader(),
                                               new Class<?>[] {requiredType}, new ObjectFactoryDelegatingInvocationHandler(factory));
    }
    else {
      return factory.getObject();
    }
  }
  return autowiringValue;
}
```

```java
private static class ObjectFactoryDelegatingInvocationHandler implements InvocationHandler, Serializable {

  private final ObjectFactory<?> objectFactory;

  public ObjectFactoryDelegatingInvocationHandler(ObjectFactory<?> objectFactory) {
    this.objectFactory = objectFactory;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    if (methodName.equals("equals")) {
      // Only consider equal when proxies are identical.
      return (proxy == args[0]);
    }
    else if (methodName.equals("hashCode")) {
      // Use hashCode of proxy.
      return System.identityHashCode(proxy);
    }
    else if (methodName.equals("toString")) {
      return this.objectFactory.toString();
    }
    try {
      // 通过objectFactory.getObject()获取实际对象，然后通过反射调用该对象的方法
      return method.invoke(this.objectFactory.getObject(), args);
    }
    catch (InvocationTargetException ex) {
      throw ex.getTargetException();
    }
  }
}
}
```

```java
private static class RequestObjectFactory implements ObjectFactory<ServletRequest>, Serializable {

  @Override
  public ServletRequest getObject() {
    // 通过ThreadLocal获取到ServletRequest对象
    return currentRequestAttributes().getRequest();
  }

  @Override
  public String toString() {
    return "Current HttpServletRequest";
  }
}
```

### 使用ObjectProvider类型进行依赖注入

#### 解决Bean不存在问题

```java
@Component
public class Home {

  private User user;

  public Home(ObjectProvider<User> user) {
    // 如果User Bean对象不存在，这里会返回null
    this.user = user.getIfAvailable();
  }
}
```

#### 解决多Bean问题

```java
@Component
public class HomeService {

 private User user;

 public Home(ObjectProvider<User> user) {
  // 如果发现多个匹配的Bean，可以依赖需求从中手动选择一个
  this.user = user.orderedStream().findFirst().orElse(null);
 }
}
```