---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring事务实现原理分析
subtitle: 
date:   2022-03-23 14:35:20 +0800
description: Spring系列-Spring事务实现原理分析
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---

上一篇文章[Spring系列-Spring事务的基本概念](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/03/22/Spring%E7%B3%BB%E5%88%97-Spring%E4%BA%8B%E5%8A%A1%E7%9A%84%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5.html)中我们介绍了事务的基本概念以及Spring在实现过程中常用类的作用。这一篇文章我们就从源码的角度来介绍以下Spring是怎么实现事务的。

## Spring使用事务的两种方式

### 编程式事务

- 通过PlatformTransactionManager控制事务，该方式我们需要手动调用commit()或rollback()方法。
- 通过TransactionTemplate控制事务，该方式是对上一种方式的封装，将通用逻辑封装起来更方便使用，我们只需将业务代码放到回调函数中即可。该方式不仅业务代理中抛出异常会自动回滚，还可以通过调用transactionStatus.setRollbackOnly()设置事务回滚。

### 声明式事务

- 在需要事务的类或方法上使用@Transaction注解标注，这也是我们最常用的方式。

## Spring事务的实现原理

因为声明式事务是我们最常用的，所以下面我们就以声明式事务为例进行代码讲解。这里我么直接采用Java配置的方式进行的配置，因为前面的文章中我们已经讲过了Xml配置的大概原理，所以之后的文章中我们就不再介绍Xml文件的配置方式了。

### Spring事务配置

在使用Spring事务之前，我们需要对Spring事务进行一些配置，因为有了这些类的帮助，才可以让我们这么方便的使用Spring事务。

```java
// 注册数据库连接池，所有的数据库连接都需要它进行管理
@Bean
public DataSource dataSource() {
  return DruidDataSourceFactory.createDataSource(properties);
}

// 数据库连接池事务管理器，用于管理事务，其持有dataSource，这里我们使用DataSourceTransactionManager
@Bean
public PlatformTransactionManager platformTransactionManager() {
  return new DataSourceTransactionManager(dataSource());
}

// 事务属性源，这里使用AnnotationTransactionAttributeSource实现类，其内部会解析类或方法上的@Transaction注解，并返回TransactionAttribute
@Bean
public TransactionAttributeSource transactionAttributeSource() {
  return new AnnotationTransactionAttributeSource();
}

// 事务拦截器，Spring事务其实就是由它拦截实现的，其持有transactionAttributeSource用来获取事务属性，并且持有platformTransactionManager用来管理事务
@Bean
public TransactionInterceptor transactionInterceptor() {
  TransactionInterceptor interceptor = new TransactionInterceptor();
  interceptor.setTransactionAttributeSource(transactionAttributeSource());
  interceptor.setTransactionManager(platformTransactionManager());
  return interceptor;
}

// 事务Advisor，之前我们讲过，MethodInterceptor类型的拦截器如果要起作用，需要将其封装为Advisor类型才可以，因为Spring默认在Aop代理时只会处理Advisor类型（不考虑扩展的
// AspectJ写法），如果要想Spring直接扫描MethodInterceptor类型,除非我们扩展Spring的功能
@Bean
public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
  BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
  advisor.setTransactionAttributeSource(transactionAttributeSource());
  advisor.setAdvice(transactionInterceptor());
  return advisor;
}
```

这样在执行需要事务的方法时，事务拦截器就会在AOP代理的连接器链中，从而实现事务的开启和提交等。下面我们就从TransactionInterceptor的invoke()方法开始讲解Spring事务的实现原理，因为这是事务开始的地方。

### invoke()

```java
// TransactionInterceptor.java
public Object invoke(MethodInvocation invocation) throws Throwable {
  // 获取targetClass，处理了实现了TargetClassAware接口或者是CGLIB代理类的情况
  Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

  // 使用事务执行
  return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
}
```

### invokeWithinTransaction()

```java
// TransactionAspectSupport.java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
                                         final InvocationCallback invocation) throws Throwable {

  // 获取TransactionAttributeSource，因为通过它我们可以获取到该方法或该类上的TransactionAttribute，这里使用实现类AnnotationTransactionAttributeSource
  TransactionAttributeSource tas = getTransactionAttributeSource();
  // 获取TransactionAttribute，其内部会通过SpringTransactionAnnotationParser类解析@Transaction注解
  final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
  // 推断事务管理器，如果我们指定了事务处理器，会返回我们制定的事务管理器，否则返回默认事务管理器
  final PlatformTransactionManager tm = determineTransactionManager(txAttr);
  // 获取一个标识，其实就是全类名.方法名
  final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);
  // 如果事务属性为null或者事务管理器不是CallbackPreferringPlatformTransactionManager类型，则直接执行Spring事务处理
  if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
    // 如果需要的话，创建一个事务，并封装为TransactionInfo
    TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
    Object retVal = null;
    try {
      // 调用链中的下一个拦截器
      retVal = invocation.proceedWithInvocation();
    }
    catch (Throwable ex) {
      // 如果出现异常进行后续处理，其中可能回滚事务，也可能是提交事务
      completeTransactionAfterThrowing(txInfo, ex);
      throw ex;
    }
    finally {
      // 清理事务信息，其实就是将该线程绑定的TransactionInfo修改为之前的TransactionInfo，如果在该事务之前存在也存在事务的话
      cleanupTransactionInfo(txInfo);
    }
    // 提交事务
    commitTransactionAfterReturning(txInfo);
    return retVal;
  }
  else {
    // CallbackPreferringPlatformTransactionManagerPlatformTransactionManager接口的扩展，公开了在事务中执行给定回调的方法。 此接口的实现者自动表达对回调的偏好，
    // 而不是编程的getTransaction、commit和rollback调用。调用代码可以检查给定的事务管理器是否实现了这个接口来选择准备一个回调而不是显式的事务分界控制。
    // 其实就是该方式可以让事务管理器不再按照模板式的几步进行事务管理，可以自定义实现了
    final ThrowableHolder throwableHolder = new ThrowableHolder();
    try {
      Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr, status -> {
        TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
        try {
          return invocation.proceedWithInvocation();
        }catch (Throwable ex) {
          if (txAttr.rollbackOn(ex)) {
            if (ex instanceof RuntimeException) {
              throw (RuntimeException) ex;
            }else {
              throw new ThrowableHolderException(ex);
            }
          }else {
            throwableHolder.throwable = ex;
            return null;
          }
        }finally {
          cleanupTransactionInfo(txInfo);
        }
      });
      if (throwableHolder.throwable != null) {
        throw throwableHolder.throwable;
      }
      return result;
    }catch (ThrowableHolderException ex) {
      throw ex.getCause();
    }catch (TransactionSystemException ex2) {
      throw ex2;
    }catch (Throwable ex2) {
      throw ex2;
    }
  }
}
```

### getTransactionAttribute()

```java
// AbstractFallbackTransactionAttributeSource.java
public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
  // 如果类是Object类型，不能使用事务
  if (method.getDeclaringClass() == Object.class) {
    return null;
  }

  // 首先从缓存中获取事务属性
  Object cacheKey = getCacheKey(method, targetClass);
  TransactionAttribute cached = this.attributeCache.get(cacheKey);
  if (cached != null) {
    if (cached == NULL_TRANSACTION_ATTRIBUTE) {
      return null;
    }
    else {
      return cached;
    }
  }
  else {
    // 缓存中没有事务属性，真正获取事务的属性，这里会解析@Transaction注解
    TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
    // 事务属性不存在，将NULL_TRANSACTION_ATTRIBUT放入缓存中，下次判断如果是该对象，则直接返回null
    if (txAttr == null) {
      this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
    }
    else {
      // 如果事务属性存在，将事务属性放入缓存中
      String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
      if (txAttr instanceof DefaultTransactionAttribute) {
        ((DefaultTransactionAttribute) txAttr).setDescriptor(methodIdentification);
      }
      if (logger.isTraceEnabled()) {
        logger.trace("Adding transactional method '" + methodIdentification + "' with attribute: " + txAttr);
      }
      this.attributeCache.put(cacheKey, txAttr);
    }
    return txAttr;
  }
}
```

### computeTransactionAttribute()

```java
// AbstractFallbackTransactionAttributeSource.java
protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
  // 只有public方法才支持事务
  if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
    return null;
  }

  // 这里面处理了两种情况：
  // 1. 如果该method被子类重写了，则返回重写后的method对象
  // 2. 如果该method是一个桥接方法，返回该桥接方法的原始方法
  Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

  // 首先查找方法上的事务注解
  TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
  if (txAttr != null) {
    return txAttr;
  }

  // 如果方法上没有事务注解，则寻找该方法所在类上的事务注解
  txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
  if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
    return txAttr;
  }
  // 如果还没找到，并且specificMethod和method不是一个对象，则从传入的method上查找事务注解
  if (specificMethod != method) {
    txAttr = findTransactionAttribute(method);
    if (txAttr != null) {
      return txAttr;
    }
    // 如果还没找到，则从传入的method所在的类上查找事务注解
    txAttr = findTransactionAttribute(method.getDeclaringClass());
    if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
      return txAttr;
    }
  }
  return null;
}
```

### determineTransactionAttribute()

上面的findTransactionAttribute()方法会调用determineTransactionAttribute(), 该方法中会调用SpringTransactionAnnotationParser的parseTransactionAnnotation()方法解析@Transaction注解。

```java
// AnnotationTransactionAttributeSource.java
protected TransactionAttribute determineTransactionAttribute(AnnotatedElement element) {
  // annotationParsers中设置了SpringTransactionAnnotationParser解析器
  for (TransactionAnnotationParser annotationParser : this.annotationParsers) {
    TransactionAttribute attr = annotationParser.parseTransactionAnnotation(element);
    if (attr != null) {
      return attr;
    }
  }
  return null;
}
```

### parseTransactionAnnotation()

```java
// SpringTransactionAnnotationParser.java
public TransactionAttribute parseTransactionAnnotation(AnnotatedElement element) {
  // 处理@Transaction注解
  AnnotationAttributes attributes = AnnotatedElementUtils.findMergedAnnotationAttributes(
    element, Transactional.class, false, false);
  if (attributes != null) {
    return parseTransactionAnnotation(attributes);
  }
  else {
    return null;
  }
}

// 将@Transaction注解中的信息封装到TransactionAttribute中
protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
  // 可以看到这里使用的就是我们上一章讲的一个实现类，该实现类可以配置失败回滚的规则
  RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();
  // 解析注解中可配置的各种属性
  Propagation propagation = attributes.getEnum("propagation");
  rbta.setPropagationBehavior(propagation.value());
  Isolation isolation = attributes.getEnum("isolation");
  rbta.setIsolationLevel(isolation.value());
  rbta.setTimeout(attributes.getNumber("timeout").intValue());
  rbta.setReadOnly(attributes.getBoolean("readOnly"));
  rbta.setQualifier(attributes.getString("value"));

  List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
  for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
    rollbackRules.add(new RollbackRuleAttribute(rbRule));
  }
  for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
    rollbackRules.add(new RollbackRuleAttribute(rbRule));
  }
  for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
    rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
  }
  for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
    rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
  }
  rbta.setRollbackRules(rollbackRules);

  return rbta;
}
```

### determineTransactionManager()

获取到了TransactionAttribute，通过该方法获取事务管理器。

```java
// TransactionAspectSupport.java
protected PlatformTransactionManager determineTransactionManager(@Nullable TransactionAttribute txAttr) {
  // 如果txAttr为空说明没有指定事务管理器，直接返回默认的
  if (txAttr == null || this.beanFactory == null) {
    return getTransactionManager();
  }
  // 如果注解上配置了使用哪个事务管理器，则返回指定的事务管理器
  String qualifier = txAttr.getQualifier();
  if (StringUtils.hasText(qualifier)) {
    return determineQualifiedTransactionManager(this.beanFactory, qualifier);
  }
  // 如果设置了transactionManagerBeanName，则返回设置的该名称的事务管理器
  else if (StringUtils.hasText(this.transactionManagerBeanName)) {
    return determineQualifiedTransactionManager(this.beanFactory, this.transactionManagerBeanName);
  }
  else {
    // 如果没有指定事务管理器，又存在事务属性，则获取默认的事务管理器
    PlatformTransactionManager defaultTransactionManager = getTransactionManager();
    if (defaultTransactionManager == null) {
      defaultTransactionManager = this.transactionManagerCache.get(DEFAULT_TRANSACTION_MANAGER_KEY);
      if (defaultTransactionManager == null) {
        defaultTransactionManager = this.beanFactory.getBean(PlatformTransactionManager.class);
        this.transactionManagerCache.putIfAbsent(
          DEFAULT_TRANSACTION_MANAGER_KEY, defaultTransactionManager);
      }
    }
    return defaultTransactionManager;
  }
}

// 在BeanFactory中获取指定名称的事务管理器，并放入缓存
private PlatformTransactionManager determineQualifiedTransactionManager(BeanFactory beanFactory, String qualifier) {
  PlatformTransactionManager txManager = this.transactionManagerCache.get(qualifier);
  if (txManager == null) {
    txManager = BeanFactoryAnnotationUtils.qualifiedBeanOfType(
      beanFactory, PlatformTransactionManager.class, qualifier);
    this.transactionManagerCache.putIfAbsent(qualifier, txManager);
  }
  return txManager;
}
```

### createTransactionIfNecessary()

获取了事务属性以及事务管理器，接下来就要创建事务了。

```java
// TransactionAspectSupport.java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
                                                       @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {
  // 如果事务没有名字，则设置一个名字，名字为全类名.方法名(前面说过)
  if (txAttr != null && txAttr.getName() == null) {
    txAttr = new DelegatingTransactionAttribute(txAttr) {
      @Override
      public String getName() {
        return joinpointIdentification;
      }
    };
  }

  TransactionStatus status = null;
  if (txAttr != null) {
    if (tm != null) {
      // 通过事务管理器获取事务，返回事务状态对象
      status = tm.getTransaction(txAttr);
    }
    else {
      if (logger.isDebugEnabled()) {
        logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                     "] because no transaction manager has been configured");
      }
    }
  }
  // 将事务相关信息封装到TransactionInfo中返回，并将TransactionInfo绑定到当前线程
  return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

### getTransaction()

```java
// DataSourceTransactionManager.java
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
  // 这里会返回一个DataSourceTransactionManager.DataSourceTransactionObject()对象，如果之前有事务，在该对象中的ConnectionHolder会持有一个数据库连接，如果之前没
  // 事务，则ConnectionHolder属性为null
  Object transaction = doGetTransaction();
  if (definition == null) {
    definition = new DefaultTransactionDefinition();
  }
  // 判断是否已经存在事务，其实就是根据上面返回的对象判断是否已有数据库连接并且事务是活动的
  if (isExistingTransaction(transaction)) {
    // 处理已经存在事务的情况，其实就是根据事务的传播行为进行不同的处理
    return handleExistingTransaction(definition, transaction, debugEnabled);
  }

  // 检查事务超时时间是否设置不正确
  if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
    throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
  }

  // 到了这里说明之前没有事务
  // 如果强制性需要存在事务，则抛出异常(因为事务不存在)
  if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
    throw new IllegalTransactionStateException(
      "No existing transaction found for transaction marked with propagation 'mandatory'");
  }
  // 下面这几种事务传播行为在不存在事务的情况下，行为一致，就是新建一个事务
  else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
           definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
           definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
    // 挂起事务，正常来说这里作用不大，因为目前没有事务
    SuspendedResourcesHolder suspendedResources = suspend(null);
    try {
      // 是否需要使用新的同步信息
      boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
      // 创建一个DefaultTransactionStatus对象
      DefaultTransactionStatus status = newTransactionStatus(
        definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
      // 该方法会获取数据库连接，设置隔离级别，设置事务手动提交等
      doBegin(transaction, definition);
      // 如果newSynchronization为true的话，设置新的事务同步管理器中的信息
      prepareSynchronization(status, definition);
      return status;
    }
    catch (RuntimeException | Error ex) {
      resume(null, suspendedResources);
      throw ex;
    }
  }
  else {
    // Create "empty" transaction: no actual transaction, but potentially synchronization.
    boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
    return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
  }
}
```

### doGetTransaction()

```java
// DataSourceTransactionManager.java
protected Object doGetTransaction() {
  DataSourceTransactionManager.DataSourceTransactionObject txObject = new DataSourceTransactionManager.DataSourceTransactionObject();
  txObject.setSavepointAllowed(this.isNestedTransactionAllowed());
  // 这里是从当前线程中获取ConnectionHolder，如果获取到说明之前存在事务，如果获取不到说明不存在事务
  ConnectionHolder conHolder = (ConnectionHolder)TransactionSynchronizationManager.getResource(this.obtainDataSource());
  // 将获取到的ConnectionHolder设置到新的txObject中，并指明这不是一个新的ConnectionHolder，因为这里拿到的是已存在的事务使用的ConnectionHolder
  txObject.setConnectionHolder(conHolder, false);
  return txObject;
}
```

### doBegin()

```java
// DataSourceTransactionManager.java
protected void doBegin(Object transaction, TransactionDefinition definition) {
  DataSourceTransactionManager.DataSourceTransactionObject txObject = (DataSourceTransactionManager.DataSourceTransactionObject)transaction;
  Connection con = null;

  try {
    // 如果不存在ConnectionHolder，或者资源与事务同步设置为true, 则需要获取一个数据库连接，并创建一个ConnectionHolder设置到txObject中
    // 设置资源与事务同步有什么用？如果当前存在事务，并且又要创建一个新的事务，此时isSynchronizedWithTransaction为true, 会在新开的事务中使用一个新的数据库连接
    if (!txObject.hasConnectionHolder() || txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
      // 通过数据库连接池获取一个数据库连接
      Connection newCon = this.obtainDataSource().getConnection();
      if (this.logger.isDebugEnabled()) {
        this.logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
      }
      txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
    }
		// 这里其实就是标记如果在该事务中还需要创建新事务的话，数据库连接啥的要重新拿一个，不要使用当前事务的
    txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
    con = txObject.getConnectionHolder().getConnection();
    // 如果是只读事务，将数据库连接设置为只读属性，如果自定义了数据库隔离级别，设置数据库连接的隔离属性，并返回之前的隔离级别
    Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
    // 保存之前的隔离级别
    txObject.setPreviousIsolationLevel(previousIsolationLevel);
    // 如果事务是自动提交的，设置手动提交事务
    if (con.getAutoCommit()) {
      // 设置一个标识，意思是事务执行完后需要将该数据库连接的事务恢复为之前的自动提交
      txObject.setMustRestoreAutoCommit(true);
      if (this.logger.isDebugEnabled()) {
        this.logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
      }
      con.setAutoCommit(false);
    }
    // 如果事务是只读的，执行 SET TRANSACTION READ ONLY 语句
    this.prepareTransactionalConnection(con, definition);
    // 将该数据库连接的事务设置为活动的
    txObject.getConnectionHolder().setTransactionActive(true);
    // 设置事务超时时间
    int timeout = this.determineTimeout(definition);
    if (timeout != -1) {
      txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
    }
    // 如果是新拿的数据库连接，这里要将新的数据库连接holder绑定到当前线程，实际上是名为resources的ThreadLocal中
    if (txObject.isNewConnectionHolder()) {
      TransactionSynchronizationManager.bindResource(this.obtainDataSource(), txObject.getConnectionHolder());
    }

  } catch (Throwable var7) {
    if (txObject.isNewConnectionHolder()) {
      // 如果出现异常，释放连接，将txObject的ConnectionHolder设置为null
      DataSourceUtils.releaseConnection(con, this.obtainDataSource());
      txObject.setConnectionHolder((ConnectionHolder)null, false);
    }

    throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", var7);
  }
}
```

### prepareSynchronization()

该方法将事务的一些信息设置到事务同步器中。

```java
// AbstractPlatformTransactionManager.java
protected void prepareSynchronization(DefaultTransactionStatus status, TransactionDefinition definition) {
  if (status.isNewSynchronization()) {
    TransactionSynchronizationManager.setActualTransactionActive(status.hasTransaction());
    TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(
      definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT ?
      definition.getIsolationLevel() : null);
    TransactionSynchronizationManager.setCurrentTransactionReadOnly(definition.isReadOnly());
    TransactionSynchronizationManager.setCurrentTransactionName(definition.getName());
    TransactionSynchronizationManager.initSynchronization();
  }
}
```

### prepareTransactionInfo()

```java
// TransactionAspectSupport.java
protected TransactionInfo prepareTransactionInfo(@Nullable PlatformTransactionManager tm,
                                                 @Nullable TransactionAttribute txAttr, String joinpointIdentification,
                                                 @Nullable TransactionStatus status) {
  TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);
  if (txAttr != null) {
    txInfo.newTransactionStatus(status);
  }
  else {
    if (logger.isTraceEnabled()) {
      logger.trace("Don't need to create transaction for [" + joinpointIdentification +
                   "]: This method isn't transactional.");
    }
  }
  // 将TransactionInfo对象绑定到当前线程
  txInfo.bindToThread();
  return txInfo;
}
```

### commitTransactionAfterReturning()

```java
// TransactionAspectSupport.java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
  if (txInfo != null && txInfo.getTransactionStatus() != null) {
    // 通过事务管理器提交事务
    txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
  }
}
```

### commit()

```java
// AbstractPlatformTransactionManager.java
public final void commit(TransactionStatus status) throws TransactionException {
  if (status.isCompleted()) {
    throw new IllegalTransactionStateException(
      "Transaction is already completed - do not call commit or rollback more than once per transaction");
  }
  DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
  // 判断事务状态是否设置了回滚，编程式事务我们可以通过设置该字段实现事务的回滚
  if (defStatus.isLocalRollbackOnly()) {
    if (defStatus.isDebug()) {
      logger.debug("Transactional code has requested rollback");
    }
    // 回滚事务
    processRollback(defStatus, false);
    return;
  }
	// 全局事务处理，是用来处理分布式事务的
  if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
    if (defStatus.isDebug()) {
      logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
    }
    processRollback(defStatus, true);
    return;
  }
  // 提交事务
  processCommit(defStatus);
}
```

### processCommit()

```java
// AbstractPlatformTransactionManager.java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
   try {
      boolean beforeCompletionInvoked = false;

      try {
         boolean unexpectedRollback = false;
         // 事务管理器事务提交前扩展点，默认没有实现
         prepareForCommit(status);
         // 事务提交前触发所有事务同步器的beforeCommit()方法
         triggerBeforeCommit(status);
         // 事务完成前触发所有事务同步器的beforeCompletion()方法
         triggerBeforeCompletion(status);
         beforeCompletionInvoked = true;
         // 如果是一个保存点，释放保存点
         if (status.hasSavepoint()) {
            if (status.isDebug()) {
               logger.debug("Releasing transaction savepoint");
            }
            unexpectedRollback = status.isGlobalRollbackOnly();
            status.releaseHeldSavepoint();
         }
         else if (status.isNewTransaction()) {
            // 如果是一个新事务，提交事务
            if (status.isDebug()) {
               logger.debug("Initiating transaction commit");
            }
            unexpectedRollback = status.isGlobalRollbackOnly();
            // 提交事务，实际就是调用connection的commit()方法
            doCommit(status);
         }
         else if (isFailEarlyOnGlobalRollbackOnly()) {
            unexpectedRollback = status.isGlobalRollbackOnly();
         }

         // 如果我们有一个仅全局回滚标记但仍然没有从提交中获得相应的异常，则抛出UnexpectedRollbackException。
         if (unexpectedRollback) {
            throw new UnexpectedRollbackException(
                  "Transaction silently rolled back because it has been marked as rollback-only");
         }
      }
      catch (UnexpectedRollbackException ex) {
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
         throw ex;
      }
      catch (TransactionException ex) {
         // 提交出现事务异常，并且设置了事务提交时候进行回滚
         if (isRollbackOnCommitFailure()) {
            // 执行事务回滚逻辑
            doRollbackOnCommitException(status, ex);
         }else {
            // 否则触发事务完成后所有事务同步器的AfterCompletion()方法
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
         }
         throw ex;
      }
      catch (RuntimeException | Error ex) {
         // 如果异常之前还没有执行所有事务同步器的BeforeCompletion()方法，这里补偿执行
         if (!beforeCompletionInvoked) {
            triggerBeforeCompletion(status);
         }
         // 如果是运行时异常或Error，执行事务回滚逻辑
         doRollbackOnCommitException(status, ex);
         throw ex;
      }
      try {
         // 事务提交后触发所有事务同步器的afterCommit()方法
         triggerAfterCommit(status);
      }
      finally {
         // 事务完成后触发所有事务同步器的AfterCompletion()方法
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
      }
   }
   finally {
      // 事务结束后的清理逻辑
      cleanupAfterCompletion(status);
   }
}
```

### cleanupAfterCompletion()

```java
// AbstractPlatformTransactionManager.java
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
  // 设置事务状态为已完成
  status.setCompleted();
  // 清空该事务的事务同步器
  if (status.isNewSynchronization()) {
    TransactionSynchronizationManager.clear();
  }
  if (status.isNewTransaction()) {
    // 这里首先从当前线程解绑该链接信息，然后恢复连接配置并释放连接
    doCleanupAfterCompletion(status.getTransaction());
  }
  if (status.getSuspendedResources() != null) {
    if (status.isDebug()) {
      logger.debug("Resuming suspended transaction after completion of inner transaction");
    }
    Object transaction = (status.hasTransaction() ? status.getTransaction() : null);
    // 如果之前挂起了事务，这里用来恢复之前挂起的事务
    resume(transaction, (SuspendedResourcesHolder) status.getSuspendedResources());
  }
}
```

### doCleanupAfterCompletion()

```java
// DataSourceTransactionManager.java
protected void doCleanupAfterCompletion(Object transaction) {
  DataSourceTransactionManager.DataSourceTransactionObject txObject = (DataSourceTransactionManager.DataSourceTransactionObject)transaction;
  if (txObject.isNewConnectionHolder()) {
    // 解绑数据库连接
    TransactionSynchronizationManager.unbindResource(this.obtainDataSource());
  }

  Connection con = txObject.getConnectionHolder().getConnection();

  try {
    // 重新将连接设置为自动提交
    if (txObject.isMustRestoreAutoCommit()) {
      con.setAutoCommit(true);
    }
		// 将连接恢复之前的隔离级别以及是否只读状态
    DataSourceUtils.resetConnectionAfterTransaction(con, txObject.getPreviousIsolationLevel());
  } catch (Throwable var5) {
    this.logger.debug("Could not reset JDBC Connection after transaction", var5);
  }

  if (txObject.isNewConnectionHolder()) {
    if (this.logger.isDebugEnabled()) {
      this.logger.debug("Releasing JDBC Connection [" + con + "] after transaction");
    }
		// 将ConnectionHolder中的currentConnection设置为null，并释放连接到连接池中
    DataSourceUtils.releaseConnection(con, this.dataSource);
  }
	// 重置ConnectionHolder的所有状态
  txObject.getConnectionHolder().clear();
}
```

至此，事务实现的大体流程我们就讲完了。其中processRollback()和processCommit()大体相似，也是执行扩展点，执行同步器的各个回调方法，并回滚事务。