---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring Aop实现原理分析
date:   2022-03-16 12:45:20 +0800
description: Spring系列-Spring Aop实现原理分析 AspectJ JDK动态代理 CGLIB动态代理
img: posts/common/spring.png
tags: [spring]
author: tengjiang
toc: yes
---

|| 该文章主要介绍了Spring Aop的底层实现原理源码解析。||

前面我们讲了Spring Aop的基本概念和几种使用方式，这一节我们讲一下Spring Aop的实现原理。其实一句话总结实现原理就是通过动态代理拦截匹配的方法调用，进行方法的增强。下面我们以

最常用的AnnotationAwareAspectJAutoProxyCreator类开始看一下Spring Aop的源码。

首先看一下AnnotationAwareAspectJAutoProxyCreator类, 该类继承的层级很深，最终会发现，它实现了BeanPostProcessor接口，之前我们也说过这个接口，这是Spring的一个扩展点，可以在Spring初始化前后进行干涉Bean的创建流程。所以AnnotationAwareAspectJAutoProxyCreator是在什么时候起作用的呢？

还记得Spring中Bean的创建流程吗？如果不记得了去回顾一下[Spring系列-Bean创建流程](https://www.tengjiang.site/Spring%E7%B3%BB%E5%88%97-Bean%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B.html)，在那篇文章中，我们讲了一个方法initializeBean(), 在该方法中会调用所有BeanPostProcessor后置处理器的postProcessAfterInitialization()方法（暂不考虑循环依赖的问题），而该方法实际上是在它的实现类AbstractAutoProxyCreator中实现的，AbstractAutoProxyCreator是Spring Aop中非常主要的一个类。

## postProcessAfterInitialization()

```java
// AbstractAutoProxyCreator.java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
  if (bean != null) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    // 这个判断和解决循环依赖有关
    if (this.earlyProxyReferences.remove(cacheKey) != bean) {
      // 该方法看是否要进行代理该Bean
      return wrapIfNecessary(bean, beanName, cacheKey);
    }
  }
  return bean;
}
```

## wrapIfNecessary()

该方法是看该Bean是否需要代理，如果需要则创建代理类，否则直接返回原Bean。

```java
// AbstractAutoProxyCreator.java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
  // 如果我们对该Bean指定了targetSource，则这里不再代理，因为如果指定了targetSource，会在postProcessBeforeInstantiation()就进行提前代理
  if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
    return bean;
  }
  // 下面的代码会根据不同的情况进行设置该advisedBeans集合
  if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
    return bean;
  }
  // 看是否是基础设施类，例如：Advice, PointCut, Advisor, AopInfrastructureBean，如果是，则跳过
  // 或者判断该bean是否是原始实例(beanName后缀为.ORIGINAL), 如果是，则跳过
  if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
  }

  // 获取针对该Bean所有符合条件的通知
  Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
  // 如果存在通知，则进行代理
  if (specificInterceptors != DO_NOT_PROXY) {
    // 将advisedBeans中的该bean设置为true, 说明什么？说明代理了一次之后我们还可以对代理类进行代理
    this.advisedBeans.put(cacheKey, Boolean.TRUE);
    // 创建代理类
    Object proxy = createProxy(
      bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
  }
  // 如果不用进行代理，将该bean在advisedBeans中设置为false
  this.advisedBeans.put(cacheKey, Boolean.FALSE);
  return bean;
}
```

## getAdvicesAndAdvisorsForBean()

```java
// AbstractAdvisorAutoProxyCreator.java
protected Object[] getAdvicesAndAdvisorsForBean(
  Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
  // 获取可以应用到该Bean的所有Advisor
  List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
  // 如果Advisor的个数为0，则说明不需要代理
  if (advisors.isEmpty()) {
    return DO_NOT_PROXY;
  }
  return advisors.toArray();
}
```

### findEligibleAdvisors()

该方法用于获取所有有资格应用于该Bean的Advisor。

```java
// AbstractAdvisorAutoProxyCreator.java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
  // 获取所有候选的Advisor，即所有的Advisor
  List<Advisor> candidateAdvisors = findCandidateAdvisors();
  // 在候选的Advisor中找到可以匹配到该Bean的Advisor
  List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
  // 扩展获取Advisor的方式，这里用来实现了处理AspectJ注解
  extendAdvisors(eligibleAdvisors);
  if (!eligibleAdvisors.isEmpty()) {
    // 对Advisor进行排序
    eligibleAdvisors = sortAdvisors(eligibleAdvisors);
  }
  return eligibleAdvisors;
}
```

### findCandidateAdvisors()

该方法在AnnotationAwareAspectJAutoProxyCreator类中进行了重写，增加了@AspectJ注解解析逻辑。该方法只会处理Advisor类型的Bean以及被@AspectJ注解标记的Bean，并不会处理实现了Advice接口或MethodInterceptor接口的Bean。如果想要实现了Advice接口的Bean生效，可以将其包装为Advisor，如果想要实现了MethodInterceptor接口的Bean生效，可以依据我们上一篇文章[Spring系列-Spring Aop使用的几种方式](https://www.tengjiang.site/Spring%E7%B3%BB%E5%88%97-Spring-Aop%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90.html)介绍的前几种方式，手动设置该拦截器。

```java
protected List<Advisor> findCandidateAdvisors() {
  // 调用父类中的findCandidateAdvisors()方法，也就是下面的实现
  List<Advisor> advisors = super.findCandidateAdvisors();
  if (this.aspectJAdvisorsBuilder != null) {
    // 解析@AspectJ相关注解，将注解方式标注的Advisor添加到集合中
    advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
  }
  return advisors;
}
```

```java
// AbstractAdvisorAutoProxyCreator.java
protected List<Advisor> findCandidateAdvisors() {
  Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
  return this.advisorRetrievalHelper.findAdvisorBeans();
}
```

### findAdvisorBeans()

```java
// BeanFactoryAdvisorRetrievalHelper.java
public List<Advisor> findAdvisorBeans() {
  // 先从缓存中拿，如果之前获取过，就不再重新获取了
  String[] advisorNames = this.cachedAdvisorBeanNames;
  if (advisorNames == null) {
    // 获取所有Advisor类型的Bean名，包括非单例的。
    advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
      this.beanFactory, Advisor.class, true, false);
    // 放到缓存中
    this.cachedAdvisorBeanNames = advisorNames;
  }
  if (advisorNames.length == 0) {
    return new ArrayList<>();
  }
  // 下面其实就是根据beanName获取所有的Advisor的Bean实例
  List<Advisor> advisors = new ArrayList<>();
  for (String name : advisorNames) {
    if (isEligibleBean(name)) {
      if (this.beanFactory.isCurrentlyInCreation(name)) {
        if (logger.isTraceEnabled()) {
          logger.trace("Skipping currently created advisor '" + name + "'");
        }
      }
      else {
        try {
          // 获取实例，并添加到advisors集合中返回
          advisors.add(this.beanFactory.getBean(name, Advisor.class));
        }
        catch (BeanCreationException ex) {
          Throwable rootCause = ex.getMostSpecificCause();
          if (rootCause instanceof BeanCurrentlyInCreationException) {
            BeanCreationException bce = (BeanCreationException) rootCause;
            String bceBeanName = bce.getBeanName();
            if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
              continue;
            }
          }
          throw ex;
        }
      }
    }
  }
  return advisors;
}
```

### findAdvisorsThatCanApply()

该方法用于从所有的Advisor中获取到可以应用到该Bean的Advisor。

```java
// AbstractAdvisorAutoProxyCreator.java
protected List<Advisor> findAdvisorsThatCanApply(
  List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

  ProxyCreationContext.setCurrentProxiedBeanName(beanName);
  try {
    // 调用AopUtils工具类的findAdvisorsThatCanApply()方法
    return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
  }
  finally {
    ProxyCreationContext.setCurrentProxiedBeanName(null);
  }
}

// AopUtils.java
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
  if (candidateAdvisors.isEmpty()) {
    return candidateAdvisors;
  }
  List<Advisor> eligibleAdvisors = new ArrayList<>();
  // 先判断IntroductionAdvisor类型的Advisor, 如果符合条件就添加到eligibleAdvisors集合中
  for (Advisor candidate : candidateAdvisors) {
    if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
      eligibleAdvisors.add(candidate);
    }
  }
  boolean hasIntroductions = !eligibleAdvisors.isEmpty();
  // 在判断非IntroductionAdvisor类型的Advisor，如果符合条件就添加到eligibleAdvisors集合中
  for (Advisor candidate : candidateAdvisors) {
    if (candidate instanceof IntroductionAdvisor) {
      // 因为IntroductionAdvisor类型上面已经处理过了，所以这里就不再处理了，直接跳过
      continue;
    }
    if (canApply(candidate, clazz, hasIntroductions)) {
      eligibleAdvisors.add(candidate);
    }
  }
  return eligibleAdvisors;
}
```

### canApply()

```java
// AopUtils.java
public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
  // 如果是IntroductionAdvisor类型，直接通过ClassFilter进行匹配，因为引介增强就是增强的类
  if (advisor instanceof IntroductionAdvisor) {
    return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
  }
  // 如果是PointcutAdvisor类型，则通过Pointcut进行校验
  else if (advisor instanceof PointcutAdvisor) {
    PointcutAdvisor pca = (PointcutAdvisor) advisor;
    return canApply(pca.getPointcut(), targetClass, hasIntroductions);
  }
  else {
    // 如果没有切入点，就假设它适用
    return true;
  }
}

// AopUtils.java
// 对PointcutAdvisor类型校验
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
  Assert.notNull(pc, "Pointcut must not be null");
  // 如果targetClass不匹配，直接false, 否则看该类里面的方法是否匹配
  if (!pc.getClassFilter().matches(targetClass)) {
    return false;
  }

  // 方法匹配
  MethodMatcher methodMatcher = pc.getMethodMatcher();
  // 如果MethodMatcher为MethodMatcher.TRUE，说明只要类匹配，任何方法都匹配
  if (methodMatcher == MethodMatcher.TRUE) {
    return true;
  }

  IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
  if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
    introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
  }

  Set<Class<?>> classes = new LinkedHashSet<>();
  if (!Proxy.isProxyClass(targetClass)) {
    classes.add(ClassUtils.getUserClass(targetClass));
  }
  classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

  for (Class<?> clazz : classes) {
    Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
    // 匹配类中的方法，只要有一个方法匹配，该Advisor就可以应用到该Bean
    for (Method method : methods) {
      if (introductionAwareMethodMatcher != null ?
          introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
          methodMatcher.matches(method, targetClass)) {
        return true;
      }
    }
  }

  return false;
}
```

### extendAdvisors()

该方法在AbstractAdvisorAutoProxyCreator中是一个空实现，留给子类进行扩展。在AspectJAwareAdvisorAutoProxyCreator中实现了该方法。该方法主要是用于添加一个特殊Advisor,将ExposeInvocationInterceptor添加在列表的开头。该特殊Advisor可以将JoinPoint暴露出来，这样像@Around环绕通知才能在方法上获取到当前JoinPoint。

```java
// AspectJAwareAdvisorAutoProxyCreator.java
protected void extendAdvisors(List<Advisor> candidateAdvisors) {
  // 该方法通过AspectJProxyUtils中的方法实现
  AspectJProxyUtils.makeAdvisorChainAspectJCapableIfNecessary(candidateAdvisors);
}
```

### makeAdvisorChainAspectJCapableIfNecessary()

ExposeInvocationInterceptor的作用是将MethodInvocation放到ThreadLocal中去，为什么需要这么做呢？
因为虽然在每个MethodInterceptor中都可以直接获取到MethodInvocation，但是通过适配的advise却没有方法入参可以传入MethodInvocation，所以如果想在advise的通知逻辑中获取MethodInvocation，就需要ExposeInvocationInterceptor提前设置。

```java
// AspectJProxyUtils.java
public static boolean makeAdvisorChainAspectJCapableIfNecessary(List<Advisor> advisors) {
  // 如果advisor集合为null，就不添加特殊的Advisor了，因为其可能不需要代理
  if (!advisors.isEmpty()) {
    boolean foundAspectJAdvice = false;
    for (Advisor advisor : advisors) {
      // 判断该advisor是否是AspectJ类型的，如果是就需要添加特殊Advisor，用来暴露JoinPoint
      if (isAspectJAdvice(advisor)) {
        foundAspectJAdvice = true;
        break;
      }
    }
    // 将特殊Advisor添加到advisors集合
    if (foundAspectJAdvice && !advisors.contains(ExposeInvocationInterceptor.ADVISOR)) {
      advisors.add(0, ExposeInvocationInterceptor.ADVISOR);
      return true;
    }
  }
  return false;
}
```

### isAspectJAdvice()

判断是不是AspectJ类型的Advisor。

```java
// AspectJProxyUtils.java
private static boolean isAspectJAdvice(Advisor advisor) {
  return (advisor instanceof InstantiationModelAwarePointcutAdvisor ||
          advisor.getAdvice() instanceof AbstractAspectJAdvice ||
          (advisor instanceof PointcutAdvisor &&
           ((PointcutAdvisor) advisor).getPointcut() instanceof AspectJExpressionPointcut));
}
```

下面我们说一下AnnotationAwareAspectJAutoProxyCreator重写的findCandidateAdvisors()方法中的buildAspectJAdvisors()方法。

### buildAspectJAdvisors()

主要用于处理添加了@AspectJ注解的类。

```java
// BeanFactoryAspectJAdvisorsBuilder.java
public List<Advisor> buildAspectJAdvisors() {
  List<String> aspectNames = this.aspectBeanNames;
  // 第一次进来，aspectNames肯定为null
  if (aspectNames == null) {
    synchronized (this) {
      aspectNames = this.aspectBeanNames;
      if (aspectNames == null) {
        List<Advisor> advisors = new ArrayList<>();
        aspectNames = new ArrayList<>();
        // 获取beanFactory中的所有beanName
        String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
          this.beanFactory, Object.class, true, false);
        for (String beanName : beanNames) {
          // 该方法实际调用的是AnnotationAwareAspectJAutoProxyCreator类中的isEligibleAspectBean方法，默认没有设置includePatterns，所以默认都是true
          if (!isEligibleBean(beanName)) {
            continue;
          }
          // 我们必须注意不要急切地实例化 bean，因为在这种情况下它们将被 Spring 容器缓存但不会被编织。
          Class<?> beanType = this.beanFactory.getType(beanName, false);
          if (beanType == null) {
            continue;
          }
          // 判断类上有没有@AspectJ注解
          if (this.advisorFactory.isAspect(beanType)) {
            aspectNames.add(beanName);
            AspectMetadata amd = new AspectMetadata(beanType, beanName);
            // 处理单例类型
            if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
              MetadataAwareAspectInstanceFactory factory =
                new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
              // 获取该类中所有的pointcut和advice，封装成advisor
              List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
              if (this.beanFactory.isSingleton(beanName)) {
                this.advisorsCache.put(beanName, classAdvisors);
              }
              else {
                this.aspectFactoryCache.put(beanName, factory);
              }
              advisors.addAll(classAdvisors);
            }
            else {
              if (this.beanFactory.isSingleton(beanName)) {
                throw new IllegalArgumentException("Bean with name '" + beanName +
                                                   "' is a singleton, but aspect instantiation model is not singleton");
              }
              // 处理非单例类型
              MetadataAwareAspectInstanceFactory factory =
                new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
              this.aspectFactoryCache.put(beanName, factory);
              advisors.addAll(this.advisorFactory.getAdvisors(factory));
            }
          }
        }
        this.aspectBeanNames = aspectNames;
        return advisors;
      }
    }
  }
  // 这里是处理缓存的情况，如果之前已经执行过上面的逻辑，直接走这里的缓存获取
  if (aspectNames.isEmpty()) {
    return Collections.emptyList();
  }
  List<Advisor> advisors = new ArrayList<>();
  for (String aspectName : aspectNames) {
    List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
    if (cachedAdvisors != null) {
      advisors.addAll(cachedAdvisors);
    }
    else {
      MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
      advisors.addAll(this.advisorFactory.getAdvisors(factory));
    }
  }
  return advisors;
}
```

## createProxy()

如果前面找到了适用于该Bean的Advisor，则需要对该Bean创建代理。而Spring创建代理又可分为JDK动态代理和CGLIB动态代理。

```java
// AbstractAutoProxyCreator.java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
                             @Nullable Object[] specificInterceptors, TargetSource targetSource) {
 
  if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
    // 该步骤是将目标beanClass设置到该Bean的BeanDefinition的atrribute中
    // key为org.springframework.aop.framework.autoproxy.AutoProxyUtils.originalTargetClass
    AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
  }
  // 通过ProxyFactory创建代理
  ProxyFactory proxyFactory = new ProxyFactory();
  proxyFactory.copyFrom(this);
  // 如果不是CGLIB动态代理，看是否需要改为CGLIB动态代理，其实就是看BeanDefinition中的preserveTargetClass属性是否为true
  if (!proxyFactory.isProxyTargetClass()) {
    if (shouldProxyTargetClass(beanClass, beanName)) {
      proxyFactory.setProxyTargetClass(true);
    }
    else {
      // 获取需要代理的类的接口，因为JDK动态代理需要实现接口
      evaluateProxyInterfaces(beanClass, proxyFactory);
    }
  }
  // 构建该Bean最终可以使用的Advisor，里面处理了我们上一节讲的通过interceptorNames设置的拦截器，并将所有拦截器多适配成了Advisor
  Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
  proxyFactory.addAdvisors(advisors);
  proxyFactory.setTargetSource(targetSource);
  customizeProxyFactory(proxyFactory);

  proxyFactory.setFrozen(this.freezeProxy);
  if (advisorsPreFiltered()) {
    proxyFactory.setPreFiltered(true);
  }
  // 创建代理
  return proxyFactory.getProxy(getProxyClassLoader());
}
```

### evaluateProxyInterfaces()

```java
// AbstractAutoProxyCreator.java
protected void evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory) {
  // 获取该beanClass上实现的所有接口，包括父类实现的接口
  Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, getProxyClassLoader());
  boolean hasReasonableProxyInterface = false;
  for (Class<?> ifc : targetInterfaces) {
    // isConfigurationCallbackInterface(): 判断是不是InitializingBean、DisposableBean、Closeable、AutoCloseable、Aware相关的接口
    // isInternalLanguageInterface(): 判断是不是内部语言相关的接口，groovy.lang.GroovyObject、.cglib.proxy.Factory、.bytebuddy.MockAccess
    // 如果不是这些相关接口，说明是可以代理的接口，将hasReasonableProxyInterface改为true
    if (!isConfigurationCallbackInterface(ifc) && !isInternalLanguageInterface(ifc) &&
        ifc.getMethods().length > 0) {
      hasReasonableProxyInterface = true;
      break;
    }
  }
  if (hasReasonableProxyInterface) {
    // 将所有接口添加到proxyFactory的集合中
    for (Class<?> ifc : targetInterfaces) {
      proxyFactory.addInterface(ifc);
    }
  }
  else {
    // 如果没有可被代理的接口，将代理方式改为CGLIB代理
    proxyFactory.setProxyTargetClass(true);
  }
}
```

### buildAdvisors()

```java
// AbstractAutoProxyCreator.java
protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
  // 解析interceptorNames设置的拦截器，并适配成Advisor
  Advisor[] commonInterceptors = resolveInterceptorNames();

  List<Object> allInterceptors = new ArrayList<>();
  // specificInterceptors是前面我们讲的流程获取到的，按之前的流程这里会都是Advisor，但是要知道我们只讲了一种实现，不同的实现可能返回的都不同
  // 所以这里可能是interceptor，可能是advice，也可能是advisor
  if (specificInterceptors != null) {
    allInterceptors.addAll(Arrays.asList(specificInterceptors));
    if (commonInterceptors.length > 0) {
      // 如果通过interceptorNames设置拦截器了，并且设置为了优先使用它们，则它们添加到集合的头部
      if (this.applyCommonInterceptorsFirst) {
        allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
      }
      else {
        allInterceptors.addAll(Arrays.asList(commonInterceptors));
      }
    }
  }
  if (logger.isTraceEnabled()) {
    int nrOfCommonInterceptors = commonInterceptors.length;
    int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
    logger.trace("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
                 " common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
  }
  // 这里主要是为了适配，无论之前是什么类型，这里都要转换为Advisor类型
  Advisor[] advisors = new Advisor[allInterceptors.size()];
  for (int i = 0; i < allInterceptors.size(); i++) {
    advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
  }
  return advisors;
}
```

### resolveInterceptorNames()

```java
// AbstractAutoProxyCreator.java
private Advisor[] resolveInterceptorNames() {
  BeanFactory bf = this.beanFactory;
  ConfigurableBeanFactory cbf = (bf instanceof ConfigurableBeanFactory ? (ConfigurableBeanFactory) bf : null);
  List<Advisor> advisors = new ArrayList<>();
  for (String beanName : this.interceptorNames) {
    if (cbf == null || !cbf.isCurrentlyInCreation(beanName)) {
      Assert.state(bf != null, "BeanFactory required for resolving interceptor names");
      // 处理interceptorNames，获取到该拦截器实例，并将其适配成Advisor类型
      Object next = bf.getBean(beanName);
      advisors.add(this.advisorAdapterRegistry.wrap(next));
    }
  }
  return advisors.toArray(new Advisor[0]);
}
```

### wrap()

这里是为了将advice类型和MethodInterceptor类型都适配成Advisor类型，经过了这一步的适配，在ProxyFactory中就可以统一以Advisor进行处理了。

```java
// DefaultAdvisorAdapterRegistry.java
public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
  if (adviceObject instanceof Advisor) {
    return (Advisor) adviceObject;
  }
  if (!(adviceObject instanceof Advice)) {
    throw new UnknownAdviceTypeException(adviceObject);
  }
  Advice advice = (Advice) adviceObject;
  if (advice instanceof MethodInterceptor) {
    // So well-known it doesn't even need an adapter.
    return new DefaultPointcutAdvisor(advice);
  }
  for (AdvisorAdapter adapter : this.adapters) {
    // Check that it is supported.
    if (adapter.supportsAdvice(advice)) {
      return new DefaultPointcutAdvisor(advice);
    }
  }
  throw new UnknownAdviceTypeException(advice);
}
```

### getProxy()

```java
// ProxyFactory.java
public Object getProxy(@Nullable ClassLoader classLoader) {
  // ProxyFactory通过AopProxy创建代理类
  return createAopProxy().getProxy(classLoader);
}

// ProxyCreatorSupport.java
protected final synchronized AopProxy createAopProxy() {
  if (!this.active) {
    activate();
  }
  // AopProxy是通过AopProxyFactory获取，默认是DefaultAopProxyFactory
  return getAopProxyFactory().createAopProxy(this);
}

// DefaultAopProxyFactory.java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
  // 如果配置了需要优化或者指定使用CGLIB动态代理或没有用户接口，都进一步走判断逻辑，否则使用JDK动态代理
  if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
    Class<?> targetClass = config.getTargetClass();
    if (targetClass == null) {
      throw new AopConfigException("TargetSource cannot determine target class: " +
                                   "Either an interface or a target is required for proxy creation.");
    }
    // 如果目标类就是接口，或者目标类是一个JDK代理类，则使用JDK动态代理
    if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
      // Spring使用JdkDynamicAopProxy实现JDK动态代理
      return new JdkDynamicAopProxy(config);
    }
    // // Spring使用ObjenesisCglibAopProxy实现CGLIB动态代理
    return new ObjenesisCglibAopProxy(config);
  }
  else {
    return new JdkDynamicAopProxy(config);
  }
}
```

### JdkDynamicAopProxy.getProxy()

JDK动态代理实现。

```java
// JdkDynamicAopProxy.java
public Object getProxy(@Nullable ClassLoader classLoader) {
  if (logger.isTraceEnabled()) {
    logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
  }
  Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
  findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
  return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

#### completeProxiedInterfaces()

该方法就是看是否需要在已有的代理接口中添加SpringProxy、Advised、DecoratingProxy接口。有了这些接口，就相当于Spring在代理类中扩展了目标类的功能，增加了一个Spring动态代理需要的功能。

```java
// AopProxyUtils.java
static Class<?>[] completeProxiedInterfaces(AdvisedSupport advised, boolean decoratingProxy) {
  Class<?>[] specifiedInterfaces = advised.getProxiedInterfaces();
  if (specifiedInterfaces.length == 0) {
    // No user-specified interfaces: check whether target class is an interface.
    Class<?> targetClass = advised.getTargetClass();
    if (targetClass != null) {
      if (targetClass.isInterface()) {
        advised.setInterfaces(targetClass);
      }
      else if (Proxy.isProxyClass(targetClass)) {
        advised.setInterfaces(targetClass.getInterfaces());
      }
      specifiedInterfaces = advised.getProxiedInterfaces();
    }
  }
  boolean addSpringProxy = !advised.isInterfaceProxied(SpringProxy.class);
  // 如果允许代理类转为Advised类型，并且目标类没有实现Advised接口，则需要添加Advised接口
  boolean addAdvised = !advised.isOpaque() && !advised.isInterfaceProxied(Advised.class);
  boolean addDecoratingProxy = (decoratingProxy && !advised.isInterfaceProxied(DecoratingProxy.class));
  int nonUserIfcCount = 0;
  if (addSpringProxy) {
    nonUserIfcCount++;
  }
  if (addAdvised) {
    nonUserIfcCount++;
  }
  if (addDecoratingProxy) {
    nonUserIfcCount++;
  }
  Class<?>[] proxiedInterfaces = new Class<?>[specifiedInterfaces.length + nonUserIfcCount];
  System.arraycopy(specifiedInterfaces, 0, proxiedInterfaces, 0, specifiedInterfaces.length);
  int index = specifiedInterfaces.length;
  if (addSpringProxy) {
    proxiedInterfaces[index] = SpringProxy.class;
    index++;
  }
  if (addAdvised) {
    proxiedInterfaces[index] = Advised.class;
    index++;
  }
  if (addDecoratingProxy) {
    proxiedInterfaces[index] = DecoratingProxy.class;
  }
  return proxiedInterfaces;
}
```

#### findDefinedEqualsAndHashCodeMethods()

该方法是为了判断在接口中是否定义了equals()和hashcode()方法，如果定义了说明了已经重新实现了这两个方法，在代理类中调用时要进行特殊处理

```java
// JdkDynamicAopProxy.java
private void findDefinedEqualsAndHashCodeMethods(Class<?>[] proxiedInterfaces) {
  for (Class<?> proxiedInterface : proxiedInterfaces) {
    Method[] methods = proxiedInterface.getDeclaredMethods();
    for (Method method : methods) {
      if (AopUtils.isEqualsMethod(method)) {
        // 定义了equals()方法
        this.equalsDefined = true;
      }
      // 定义了hashcode()方法
      if (AopUtils.isHashCodeMethod(method)) {
        this.hashCodeDefined = true;
      }
      if (this.equalsDefined && this.hashCodeDefined) {
        return;
      }
    }
  }
}
```

#### invoke()

JDK动态代理需要实现InvocationHandler接口，在该接口定义的invoke()方法中实现代理逻辑，因为JdkDynamicAopProxy已经实现了InvocationHandler接口，所以invoke()方法实现就在JdkDynamicAopProxy类中。

```java
// JdkDynamicAopProxy.java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  Object oldProxy = null;
  boolean setProxyContext = false;
  // Spring中无论是JDK动态代理还是CGLIB动态代理，都持有targetSource，这个类中保存了我们代理的目标Bean，为什么通过TargetSource封装一层呢？
  // 因为通过TargetSource封装后，我们可以在代理后动态的改变targetSource中持有的目标Bean，使我们的代理更加灵活
  TargetSource targetSource = this.advised.targetSource;
  Object target = null;

  try {
    if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
      // 如果目标类本身没有定义equals()方法，则走JdkDynamicAopProxy中重写的equals()方法
      return equals(args[0]);
    }
    else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
      // 如果目标类本身没有定义hashcode()方法，则走JdkDynamicAopProxy中重写的hashcode()方法
      return hashCode();
    }
    else if (method.getDeclaringClass() == DecoratingProxy.class) {
      // 调用getDecoratedClass()方法，这个方法是DecoratingProxy接口定义的，这个接口是前面说的Spring添加进来的接口，并不是目标类本身存在的，作用就是为了获取代理的目标类
      // 本身这个方法没有实现，通过ultimateTargetClass获取该代理的实际目标类型
      return AopProxyUtils.ultimateTargetClass(this.advised);
    }
    else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
             method.getDeclaringClass().isAssignableFrom(Advised.class)) {
      // 如果该方法定义是在Advised接口中的，并且允许代理类转换为Advised接口，那么直接反射调用目标方法
      // 当然我们的目标类中肯定没有实现该方法，那么为什么还能调用到方法实现呢？因为这里的this.advised实际上是ProxyFactory, ProxyFactory中持有targetSource，
      // targetSource中持有我们的Bean，而ProxyFactory继承了AdvisedSupport类，这个类实现了Advised接口并实现了其中的方法(父类中实现)
      return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
    }

    Object retVal;

    if (this.advised.exposeProxy) {
      // 是否需要暴露代理类，如果需要，将当前代理类放到ThreadLocal中，我们可以通过AopContext.currentProxy()获取到该代理了
      oldProxy = AopContext.setCurrentProxy(proxy);
      setProxyContext = true;
    }

    // 获取目标类实例
    target = targetSource.getTarget();
    Class<?> targetClass = (target != null ? target.getClass() : null);

    // 获取适配该方法的所有拦截器
    // 注意这里获取的拦截器是根据上面的Advisor转换来的，其中包括静态拦截器和动态拦截器，动态拦截器需要在方法调用时才能判断出该拦截器适不适用
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

    // 如果拦截器链为null，说明不需要拦截，直接反射调用该方法
    if (chain.isEmpty()) {
      // We can skip creating a MethodInvocation: just invoke the target directly
      // Note that the final invoker must be an InvokerInterceptor so we know it does
      // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
      Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
      retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
    }
    else {
      // 否则说明需要拦截，将拦截器链等信息封装到ReflectiveMethodInvocation中，处理拦截器链的调用
      MethodInvocation invocation =
        new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
      // 触发拦截器链的执行
      retVal = invocation.proceed();
    }

    // 处理返回值
    Class<?> returnType = method.getReturnType();
    if (retVal != null && retVal == target &&
        returnType != Object.class && returnType.isInstance(proxy) &&
        !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
      // 特殊情况：它返回“this”并且方法的返回类型是类型兼容的。请注意，如果目标在另一个返回的对象中设置对自身的引用，我们将无能为力。
      retVal = proxy;
    }
    else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
      throw new AopInvocationException(
        "Null return value from advice does not match primitive return type for: " + method);
    }
    return retVal;
  }
  finally {
    if (target != null && !targetSource.isStatic()) {
      // Must have come from TargetSource.
      targetSource.releaseTarget(target);
    }
    if (setProxyContext) {
      // Restore old proxy.
      AopContext.setCurrentProxy(oldProxy);
    }
  }
}
```

#### proceed()

```java
// ReflectiveMethodInvocation.java
public Object proceed() throws Throwable {
  // 判断如果当前拦截器已经是最后一个了，直接通过反射调用目标类方法
  if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
    return invokeJoinpoint();
  }
  // 获取指定索引的拦截器，并执行拦截器（索引会自增，所以下次进来获取到的就是下一个拦截器了）
  Object interceptorOrInterceptionAdvice =
    this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
  // 如果是动态拦截器，需要在执行之前进行匹配，如果可以应用于该方法，则执行该拦截器
  if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
    InterceptorAndDynamicMethodMatcher dm =
      (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
    Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
    // 匹配成功，则执行
    if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
      return dm.interceptor.invoke(this);
    }
    else {
      // 如果动态拦截器匹配不成功，直接执行下一个拦截器
      return proceed();
    }
  }
  else {
    // 如果不是动态拦截器，调用拦截器的invoke()方法，invoke()方法会在调回该proceed()方法
    return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
  }
}
```

### CglibAopProxy.getProxy()

CGLIB动态代理实现。

```java
// CglibAopProxy.java
public Object getProxy(@Nullable ClassLoader classLoader) {
  if (logger.isTraceEnabled()) {
    logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
  }

  try {
    // 先拿到需要代理的目标类，因为CGLIB代理是依据于创建子类的方式进行代理
    Class<?> rootClass = this.advised.getTargetClass();
    Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

    Class<?> proxySuperClass = rootClass;
    // 这里是判断，当前要代理的类是否本身就是CGLIB代理类(判断是否包含$$)，如果是，获取该类的父类，因为其父类就是真正需要代理的类
    // 并且获取被代理实现的接口，并将接口保存起来
    if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
      proxySuperClass = rootClass.getSuperclass();
      Class<?>[] additionalInterfaces = rootClass.getInterfaces();
      for (Class<?> additionalInterface : additionalInterfaces) {
        this.advised.addInterface(additionalInterface);
      }
    }

    // 校验该类，将不符合代理要求的方法信息打印出来进行提示
    validateClassIfNecessary(proxySuperClass, classLoader);

    // CGLIB通过Enhancer对象进行创建代理对象
    Enhancer enhancer = createEnhancer();
    if (classLoader != null) {
      enhancer.setClassLoader(classLoader);
      if (classLoader instanceof SmartClassLoader &&
          ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
        enhancer.setUseCache(false);
      }
    }
    // 设置需要继承的父类，也就是要被代理的目标类
    enhancer.setSuperclass(proxySuperClass);
    // 设置接口信息，与JDK动态代理不同的是，这里不会设置DecoratingProxy接口
    enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
    // 设置代理类命名策略
    enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
    // 设置用于从此生成器创建字节码的策略
    enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));
    
    // 这个方法非常重要，因为代理类要执行的拦截器就是在这个方法中获取的
    Callback[] callbacks = getCallbacks(rootClass);
    Class<?>[] types = new Class<?>[callbacks.length];
    for (int x = 0; x < types.length; x++) {
      types[x] = callbacks[x].getClass();
    }
    // 设置callback过滤器，其内部封装了在什么情况下使用什么callback
    enhancer.setCallbackFilter(new ProxyCallbackFilter(
      this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
    // 设置callback的类型
    enhancer.setCallbackTypes(types);

    // 创建CGLIB代理对象
    return createProxyClassAndInstance(enhancer, callbacks);
  }
  catch (CodeGenerationException | IllegalArgumentException ex) {
    throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
                                 ": Common causes of this problem include using a final class or a non-visible class",
                                 ex);
  }
  catch (Throwable ex) {
    // TargetSource.getTarget() failed
    throw new AopConfigException("Unexpected AOP exception", ex);
  }
}
```

#### getCallbacks()

```java
// CglibAopProxy.java
private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
  // 是否需要暴露代理对象
  boolean exposeProxy = this.advised.isExposeProxy();
  // 配置是否被冻结
  boolean isFrozen = this.advised.isFrozen();
  // 是否是静态的，也就是targetSource中的target是否是不变的，如果是，那就是静态的
  boolean isStatic = this.advised.getTargetSource().isStatic();

  // 封装AOP的拦截器，最终底层还是JDK那一套代码
  Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

  // 这里分情况进行处理
  Callback targetInterceptor;
  // 看是否需要暴露代理对象
  if (exposeProxy) {
    // 需要暴露代理对象，这里的这两个interceptor与下面的区别就是有没有在调用目标方法之前调用AopContext.setCurrentProxy(proxy)设置代理对象
    targetInterceptor = (isStatic ?
                         new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
                         new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource()));
  }
  else {
    targetInterceptor = (isStatic ?
                         // 如果target是不变的，那么这里直接设置target对象即可
                         new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
                         // 如果target是变化的，那么这里设置targetSource对象，每次都在执行目标对象方法之前实时从targetSource中getTarget()
                         new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
  }

  // 静态目标的调度程序。 Dispatcher比拦截器快得多。只要可以确定方法肯定不返回“this”，就会使用它。只对不变的target有效
  Callback targetDispatcher = (isStatic ?
                               new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp());

  // 封装所有的拦截器，什么情况下使用什么interceptor，是依靠CallbackFilter返回的callback数组的索引确定的，CallbackFilter源码请看下面
  Callback[] mainCallbacks = new Callback[] {
    aopInterceptor,  // for normal advice
    targetInterceptor,  // invoke target without considering advice, if optimized
    new SerializableNoOp(),  // no override for methods mapped to this
    targetDispatcher, this.advisedDispatcher,
    new EqualsInterceptor(this.advised),
    new HashCodeInterceptor(this.advised)
  };

  Callback[] callbacks;

  // 其实上面的代理已经可以完成CGLIB动态代理的功能了，下面的代理是为了性能优化
  // 如果target对象不变并且配置被冻结了就可以进行优化了
  // 这里的优化就是将该被代理类的所有方法对应的拦截器链都进行一一映射保存起来，这么在执行方法调用的时候，就不用每次都获取了，因为在这种情况下信息都确定不会改变了，
  // 这样每个方法对应哪些拦截器都已经确定了
  if (isStatic && isFrozen) {
    // 获取该被代理类所有的方法
    Method[] methods = rootClass.getMethods();
    Callback[] fixedCallbacks = new Callback[methods.length];
    this.fixedInterceptorMap = new HashMap<>(methods.length);

    // 对每一个方法都找到该方法对应的拦截器链进行一一映射保存，便于调用方法时进行获取拦截器链
    for (int x = 0; x < methods.length; x++) {
      Method method = methods[x];
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, rootClass);
      fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
        chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
      this.fixedInterceptorMap.put(method, x);
    }

    // 将未优化和优化后的所有callback都放到一块
    callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
    System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
    System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
    // 标记优化后的callback的开始偏移量是在哪里
    this.fixedInterceptorOffset = mainCallbacks.length;
  }
  else {
    // 如果不能优化，直接将mainCallbacks赋值给callbacks
    callbacks = mainCallbacks;
  }
  return callbacks;
}
```

#### ProxyCallbackFilter.accept()

CallbackFilter算是一个转换器，用来对什么情况下执行什么callback进行转换。

```java
// ProxyCallbackFilter.java
// private static final int AOP_PROXY = 0;
// private static final int INVOKE_TARGET = 1;
// private static final int NO_OVERRIDE = 2;
// private static final int DISPATCH_TARGET = 3;
// private static final int DISPATCH_ADVISED = 4;
// private static final int INVOKE_EQUALS = 5;
// private static final int INVOKE_HASHCODE = 6;
public int accept(Method method) {
  // 如果方法是final的，什么都不做，不支持final类型的方法代理
  if (AopUtils.isFinalizeMethod(method)) {
    logger.trace("Found finalize() method - using NO_OVERRIDE");
    return NO_OVERRIDE;
  }
  // 如果是透明的(即可以强转并进行方法调用)，并且方法属于Advised接口，调用advisedDispatcher返回advised实例
  if (!this.advised.isOpaque() && method.getDeclaringClass().isInterface() &&
      method.getDeclaringClass().isAssignableFrom(Advised.class)) {
    if (logger.isTraceEnabled()) {
      logger.trace("Method is declared on Advised interface: " + method);
    }
    return DISPATCH_ADVISED;
  }
  // 如果是equals()方法，调用EqualsInterceptor
  if (AopUtils.isEqualsMethod(method)) {
    if (logger.isTraceEnabled()) {
      logger.trace("Found 'equals' method: " + method);
    }
    return INVOKE_EQUALS;
  }
  // 如果是hashcode(), 调用HashCodeInterceptor
  if (AopUtils.isHashCodeMethod(method)) {
    if (logger.isTraceEnabled()) {
      logger.trace("Found 'hashCode' method: " + method);
    }
    return INVOKE_HASHCODE;
  }
  Class<?> targetClass = this.advised.getTargetClass();
  // Proxy is not yet available, but that shouldn't matter.
  List<?> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
  boolean haveAdvice = !chain.isEmpty();
  boolean exposeProxy = this.advised.isExposeProxy();
  boolean isStatic = this.advised.getTargetSource().isStatic();
  boolean isFrozen = this.advised.isFrozen();
  // 如果存在拦截器或者配置没被冻结(没被冻结说明可能被修改)，就需要进行拦截
  if (haveAdvice || !isFrozen) {
    // 如果需要暴露代理对象，必须使用aopInterceptor
    if (exposeProxy) {
      if (logger.isTraceEnabled()) {
        logger.trace("Must expose proxy on advised method: " + method);
      }
      return AOP_PROXY;
    }
    // 如果是静态的并且冻结的，并且优化的映射集合中存在该方法，则走优化的fixedInterceptor
    if (isStatic && isFrozen && this.fixedInterceptorMap.containsKey(method)) {
      if (logger.isTraceEnabled()) {
        logger.trace("Method has advice and optimizations are enabled: " + method);
      }
      int index = this.fixedInterceptorMap.get(method);
      // 找到该方法对应的优化的fixedInterceptor在哪个位置
      return (index + this.fixedInterceptorOffset);
    }
    else {
      if (logger.isTraceEnabled()) {
        logger.trace("Unable to apply any optimizations to advised method: " + method);
      }
      // 不能使用优化，则使用aopInterceptor
      return AOP_PROXY;
    }
  }
  else {
    // 如果需要暴露代理或者不是静态的target，则使用targetInterceptor
    if (exposeProxy || !isStatic) {
      return INVOKE_TARGET;
    }
    // 如果返回值是this，则使用targetInterceptor
    Class<?> returnType = method.getReturnType();
    if (targetClass != null && returnType.isAssignableFrom(targetClass)) {
      if (logger.isTraceEnabled()) {
        logger.trace("Method return type is assignable from target type and " +
                     "may therefore return 'this' - using INVOKE_TARGET: " + method);
      }
      return INVOKE_TARGET;
    }
    else {
      if (logger.isTraceEnabled()) {
        logger.trace("Method return type ensures 'this' cannot be returned - " +
                     "using DISPATCH_TARGET: " + method);
      }
      // 到了这里就是既不需要暴露代理，又是静态的target，则调用StaticDispatcher
      return DISPATCH_TARGET;
    }
  }
}
```

到这里，Spring Aop实现源码的重点逻辑我们就讲完了。

不知道大家有没有一个疑问？Spring Aop在处理的过程中为什么要先将MethodInterceptor/Advise都适配成Advisor(通知者)，然后在最后再通过Advisor适配成MethodInterceptor去使用，来回转换的意义在哪?
先适配成Advisor是因为Advisor即可以持有MethodInterceptor/Advise又可以持有Pointcut信息，而MethodInterceptor/Advise没有Pointcut进行判断的相关信息，通过Pointcut可以进行判断该拦截器是否需要应用到该方法上，如果判断通过了，就通过该Advisor拿到MethodInterceptor去使用，因为拦截器的逻辑都是通过methodInterceptor的invoke方法去执行的。