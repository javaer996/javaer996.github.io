---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring循环依赖详解
subtitle: 
date:   2022-03-25 10:22:20 +0800
description: Spring 循环依赖 三级缓存
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---

## 什么是循环依赖?

循环依赖其实就是在我们的程序中，类A需要依赖于类B，类B又需要依赖于类A，这时候两个类实例就构成了相互依赖，即循环依赖。如下所示：


```java
@Component
public class A {
    @Autowired
    private B b;
}

@Component
public class B {
    @Autowired
    private A a;
}
```

## Spring什么情况下可以帮助我们解决循环依赖？

在spring中，如果我们的bean是单例的，并且不是通过构造方法进行依赖注入，spring大部分情况下是可以帮助我么解决循环依赖问题的。当然这是基于我们没有错误的扩展Spring功能的情况下。

## Spring是怎么解决循环依赖的？

Spring解决循环依赖的原理就是提前暴露创建中的Bean对象，如果发现循环依赖，需要的话提前进行代理对象生成，并在最后判断最终的Bean对象和提前注入的Bean对象是否是同一个，如果不是同一个则报错，这说明在注入之后该注入Bean对象又发生了改变。

Spring在解决循环依赖的过程中，使用了三级缓存：

- singletonObjects：一级缓存，用来保存已经执行完所有生命周期的单例Bean对象。
- earlySingletonObjects：二级缓存，存放提前暴露还没有最终完成的单例对象的，此时它还不能算是单例bean对象
- singletonFactories：三级缓存，用来保存单例Bean工厂方法，通过该工厂方法，可以提前暴露Bean并在需要的时候提前生成代理对象。

下面我们看一下解决循环依赖的重点代码：

首先看一下doGetBean()中的getSingletion()方法, 因为在创建Bean对象之前，都会尝试从缓存中先获取一下该Bean对象。

```java
// AbstractBeanFactory.java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

  final String beanName = transformedBeanName(name);
  Object bean;
  // 1. 先尝试从缓存中获取，这里允许从第三级缓冲获取
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }
  // ...
  else {
    if (mbd.isSingleton()) {
      // 在该带回调函数的getSingleton()中的beforeSingletonCreation()方法中，会将该beanName添加到singletonsCurrentlyInCreation集合中，该集合会在从三级缓存获取
      // Bean对象时用来判断当前Bean是否正在创建中
      sharedInstance = getSingleton(beanName, () -> {
        try {
          // 2. 如果缓存中没有获取到，则走创建Bean的逻辑
          return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
          destroySingleton(beanName);
          throw ex;
        }
      });
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }
    // ...
    return (T) bean;
  }
```
将beanName添加到标识Bean正在创建中的集合中。

```java
// DefaultSingletonBeanRegistry.java
protected void beforeSingletonCreation(String beanName) {
  // add添加
  if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
  }
}
```

先从三级缓存中获取。

```java
// DefaultSingletonBeanRegistry.java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  // 1. 从一级缓存中获取
  Object singletonObject = this.singletonObjects.get(beanName);
  // 如果一级缓存中没有并且该Bean正在创建中
  if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    synchronized (this.singletonObjects) {
      // 2. 从二级缓存中获取
      singletonObject = this.earlySingletonObjects.get(beanName);
      // 如果二级缓存中没有，并且允许循环依赖
      if (singletonObject == null && allowEarlyReference) {
        // 3. 从三级缓存中获取
        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
        if (singletonFactory != null) {
          // 如果三级缓存不为空，通过三级缓存获取不完整的Bean对象(这里其实会调用SmartInstantiationAwareBeanPostProcessor中的getEarlyBeanReference()方法)
          singletonObject = singletonFactory.getObject();
          // 将通过三级缓存获取到的不完整的Bean对象放入二级缓存中
          this.earlySingletonObjects.put(beanName, singletonObject);
          // 清除该Bean在三级缓存中的工厂方法
          this.singletonFactories.remove(beanName);
        }
      }
    }
  }
  return singletonObject;
}
```

如果通过三级缓存没有获取到Bean对象，说明需要创建Bean，此时会走creatBean()去创建Bean，实际执行的是doCreateBean()。

```java
// AbstractAutowireCapableBeanFactory.java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
  throws BeanCreationException {
  // ...

  // 如果允许循环依赖，并且当前bean正在创建中，那么先放入singletonFactories,singletonFactories属于是一个缓存，spring获取bean
  // bean实例的时候，会先从singletonObjects获取，获取不到再从earlySingletonObjects获取，如果还没有，就会从singletonFactories
  // 进行获取，因为提前暴露的对象我们已经add到singletonFactories中了，那么从singletonFactories获取对象时就会调用
  // () -> getEarlyBeanReference(beanName, mbd, bean),从而就可以提前获取到代理对象了。
  // 虽然通过三级缓存提前获取到Bean对象了，但是此时还不完整，所以该Bean后续还会继续走后面的依赖注入初始化逻辑。
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                    isSingletonCurrentlyInCreation(beanName));
  // 第一次判断：如果允许提前暴露，则将bean放入singletonFacotries中, 在循环依赖的时候，可以通过getEarlyBeanReference提前自动代理，然后在其他bean中注入提前代理的对象
  if (earlySingletonExposure) {
    if (logger.isTraceEnabled()) {
      logger.trace("Eagerly caching bean '" + beanName +
                   "' to allow for resolving potential circular references");
    }
    // 这里是重点，里面封装了能够正确注入代理的逻辑
    // 重点就是getEarlyBeanReference(beanName, mbd, bean)方法
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }

  Object exposedObject = bean;
  try {
    // 在这个方法里面，会执行AutowiredAnnotationBeanPostProcessor的postProcessProperties，从而进行了@Autowird属性的注入, 注入就可能会出现循环依赖
    populateBean(beanName, mbd, instanceWrapper);
    exposedObject = initializeBean(beanName, exposedObject, mbd);
  }catch (Throwable ex) {
    ...
  }
  // 第二次判断，主要是为了防止提前暴露的bean和最终实际的bean不一致，如果我们没有自定义或者引入会创建代理对象的BeanPostProcessor的话，是不会有问题的，因为在spring的
  // AbstractAutoProxyCreator中，如果已经通过getEarlyBeanReference提前创建过代理对象了，那么在postProcessAfterInitialization中会直接跳过代理对象的创建, 然后将
  // 原bean对象返回，所以下面的exposedObject == bean是true, 又因为在postProcessAfterInitialization跳过了自动代理类的创建，但是在getEarlyBeanReference中代理了，
  // 所以要将earlySingletonReference赋给exposedObject，保证返回的bean和提前代理并注入的bean是一致的。
  // 所以如果我们如果想自定义BeanPostProcessor并要代理bean对象的话，需要按照AbstractAutoProxyCreator处理循环依赖的模式去实现，既要实现getEarlyBeanReference，还要
  // 实现postProcessAfterInitialization，如果调用过了getEarlyBeanReference，提前暴露了bean对象，那么在postProcessAfterInitialization中要跳过
  // getEarlyBeanReference中的实现逻辑，而且不要改变bean对象。
  if (earlySingletonExposure) {
    // 获取获取提前暴露的Bean，这里不允许从第三级缓存中获取
    Object earlySingletonReference = getSingleton(beanName, false);
    // 如果提前暴露对象为null，说明没有提前暴露Bean对象，也就没有发生循环依赖，如果不为null，说明发生循环依赖了
    if (earlySingletonReference != null) {
      // 如果exposedObject与bean相等，说明经过了Bean的整个创建周期后，Bean对象没有发生变化，此时将提前暴露的Bean赋值给最终Bean。
      if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
      }
      // 走到这里说明在提前暴露Bean之后，在后续逻辑中最终Bean又被修改了
      // 如果不允许在出现循环依赖，最终是代理对象的情况下注入原始Bean并且依赖该Bean的Bean已经被创建了，则报错
      else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        String[] dependentBeans = getDependentBeans(beanName);
        Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
        for (String dependentBean : dependentBeans) {
          if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
            actualDependentBeans.add(dependentBean);
          }
        }
        // 这里就是报错的地方，因为检测出来了依赖的bean和实际的bean不一致
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
  ...
  return exposedObject;
}
```

第三级缓存实际上调用的是这里。

```java
// AbstractAutowireCapableBeanFactory.java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
  Object exposedObject = bean;
  if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
      // 从这里就可以发现，对于提前暴露的bean引用，这里会调用SmartInstantiationAwareBeanPostProcessor的
      //getEarlyBeanReference的方法，并将该方法返回的对象作为实际暴露的对象，那么就不难理解为什么我们重写了这个方法后，就可以将我们代
      //理后的对象正确注入了
      if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
        SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
        exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
      }
    }
  }
  return exposedObject;
}
```

## Prototype类型为什么解决不了循环依赖？

因为Prototype类型的Bean在每次获取时都要重新创建，而依赖注入的Bean在注入后就结束了，不能再修改了，所以Prototype不能解决循环依赖。在spring的AbstractBeanFactory中的doGetBean方法中有一个判断，如果是Prototype类型并且该实例正在创建中，直接抛出异常。

```java
// AbstractBeanFactory.java -> doGetBean()
if (isPrototypeCurrentlyInCreation(beanName)) {
	throw new BeanCurrentlyInCreationException(beanName);
}

protected boolean isPrototypeCurrentlyInCreation(String beanName) {
	Object curVal = this.prototypesCurrentlyInCreation.get();
	return (curVal != null &&
			(curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
}
```

## 构造方法注入为什么解决不了循环依赖？

思考一个逻辑，Spring创建A的时候，发现A的构造函数中需要注入B，然后spring去创建B，这时候发现，B的构造函数中又需要注入A，可是这时候A还在等待B创建成功后进行注入，A还没创建好呢，这时候就构成了死等待，A等待B创建，B等待A创建，最终的结果就是都进行不下去了，spring检测到后就只能给我们抛异常了。

## 自定义BeanPostProcessor并支持循环依赖

如果我们想要自定义BeanPostProcessor，并且可能需要通过代理原始类，扩展我们的功能，那么这时候，如果我们像下面这样写：

```java
@Component
public class TestBeanPostProcessor implements BeanPostProcessor, MethodInterceptor{

  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    return bean;
  }

  @Override
  public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if(bean instanceof A){
      System.out.println("对A创建代理");
      Enhancer enhancer = new Enhancer();
      enhancer.setSuperclass(A.class);
      enhancer.setCallback(this);
      return enhancer.create();
    }
    return bean;
  }

  @Override
  public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
    System.out.println("代理拦截" + method.getName());
    Object object = methodProxy.invokeSuper(o, objects);
    return object;
  }
}
```
这样写会出现如下错误：


```java
org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'a': Bean with name 
'a' has been injected into other beans [b] in its raw version as part of a circular reference, but has eventually been 
wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-
eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.
```
这段错误大概的意思就是，我们已经注入的bean后来被修改了，导致注入的bean和最终实际的bean不是一个，所以报错了，当然，你也可以按照提示，通过allowEagerInit这个属性进行配置，允许这种不一致的情况发生，只是我不推荐这种做法，很有可能带来莫名其妙的错误。

下面我们来改写一下实现方式：


```java
@Component
public class TestBeanPostProcessor implements SmartInstantiationAwareBeanPostProcessor,MethodInterceptor{

  private List<String> alreadyWrap = new ArrayList();

  @Override
  public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    if(bean instanceof A){
      alreadyWrap.add(beanName);
      System.out.println("对A创建代理");
      Enhancer enhancer = new Enhancer();
      enhancer.setSuperclass(A.class);
      enhancer.setCallback(this);
      return enhancer.create();
    }
    return bean;
  }
  
  @Override
  public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if(bean instanceof A){
      // 如果已经在提前暴露对象的时候创建代理对象了，这里就不再进行重复代理了
      if (!alreadyWrap.contains(beanName)) {
        System.out.println("对A创建代理");
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(A.class);
        enhancer.setCallback(this);
        return enhancer.create();
      }
    }
    return bean;
  }

  @Override
  public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
    System.out.println("代理拦截" + method.getName());
    Object object = methodProxy.invokeSuper(o, objects);
    return object;
  }
}
```
你会发现这种方式成功启动了，并且在B中注入的A实例就是我们代理后的对象。

注意一下这两种实现方式：

第一种是实现了BeanPostProcessor，并对postProcessAfterInitialization()方法进行了重写；

第二种是实现了SmartInstantiationAwareBeanPostProcessor，并对getEarlyBeanReference进行了重写。

要想明白原因，我们需要知道Spring在什么时候会触发BeanPostProcessor的调用。之所以第二种能够成功，是因为Spring中提前暴露bean时的一段处理逻辑，这段逻辑位于AbstractAutowireCapableBeanFactory中的doCreateBean方法中，上面已经讲过了。

现在我们来看一看第一种方式为什么不能成功?

通过上面代码的标注，我们知道了第一种方式postProcessAfterInitialization()方法的调用是在initializeBean()中，而我们的@Autowird注解的注入是在initializeBean的前一个方法populateBean中，也就是B中都已经注入了A提前暴露出来的对象后，我们的postProcessAfterInitialization()方法又将A对象实例给改成了我们的代理对象，从而导致了注入的A的bean对象和最终的A的bean对象不是一个。