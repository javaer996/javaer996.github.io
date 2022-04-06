---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring事务的基本概念
subtitle: 多一分心力去注意别人、就少一分心力反省自己。
date:   2022-03-22 15:45:20 +0800
description: Spring系列-Spring事务的基本概念
categories: [Spring系列]
tags: [spring]
author: tengjiang
toc: yes
---

## 什么是事务

一般我们说事务就是在说数据库事务，数据库事务是访问并可能操作各种数据项的一个数据库操作序列，这些操作要么全部执行，要么全部不执行，是一个不可分割的工作单位。

## 事务的四个特性

### 原子性（Atomicity）

一个事务中的所有操作，或者全部完成，或者全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚到事务开始前的状态，就像这个事务从来没有执行过一样。

### 一致性（Consistency）

在事务开始之前和事务结束以后，数据库的完整性没有被破坏。

### 隔离性（Isolation）

数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括未提交读（Read uncommitted）、提交读（read committed）、可重复读（repeatable read）和串行化（Serializable）。

### 持久性（Durability）

事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。

## 事务的隔离级别

### 未提交读（Read uncommitted）

一个事务可以读取另一个未提交事务的数据，什么都无法保证(最低隔离级别)。

### 提交读（read committed）

一个事务要等另一个事务提交后才能读取数据，可避免脏读。

### 可重复读（repeatable read）

一个事务中不可以读取到另一个事务中修改的数据，多次读取同一条数据数据内容不发生改变(Mysql默认隔离级别)。

### 串行化（Serializable）

最高的事务隔离级别，在该级别下，事务串行化顺序执行，可以避免脏读、不可重复读与幻读。

## Spring事务的传播行为

### REQUIRED(Spring默认的事务传播行为)

如果当前没有事务，则自己新建一个事务，如果当前存在事务，则加入这个事务。

### SUPPORTS

当前存在事务，则加入当前事务，如果当前没有事务，就以非事务方法执行。

### MANDATORY

当前存在事务，则加入当前事务，如果当前事务不存在，则抛出异常。

### REQUIRES_NEW

创建一个新事务，如果存在当前事务，则挂起该事务。

### NOT_SUPPORTED

始终以非事务方式执行,如果当前存在事务，则挂起当前事务。

### NEVER

不使用事务，如果当前事务存在，则抛出异常。

### NESTED

如果当前事务存在，则在嵌套事务中执行，否则REQUIRED的操作一样 (如果存在嵌套事务，父事务回滚，子事务也会回滚)。

## Spring中事务的基本概念

### Connection

数据库连接接口定义，用来操作数据库，其定义了获取Statement，设置autoCommit，commit, rollback, close等数据库操作。

### ConnectionHolder

数据库连接持有者，其内部保存了一个数据库连接，并保存有该数据库连接是否存在事务，是否支持savepoint等信息。这是Spring的一贯写法，将某个资源使用holder封装，并在holder中支持一些对该资源的操作。

### PlatformTransactionManager

事务管理器接口，是Spring对事务管理器的一个高级抽象接口，定义了获取事务，提交事务，回滚事务的基本接口。不同平台想要集成Spring事务，都要实现该接口，实现自己的事务管理器。

### DataSourceTransactionManager

该类是JDBC针对PlatformTransactionManager接口的实现类，除了支持获取事务，提交事务，回滚事务这些基本功能接口，还支持挂起事务，恢复事务，资源清理等功能。

### DataSourceTransactionManager.DataSourceTransactionObject

该类是DataSourceTransactionManager的一个内部静态类，其继承了JdbcTransactionObjectSupport抽象类，辅助DataSourceTransactionManager类进行一些事务处理。该类持有ConnectionHolder，并记录该holder是否是一个新的，如果是true，表明该数据库连接是在一个新事务中，如果是false，则说明该数据库连接是在一个老事务中。

### SuspendedResourcesHolder

该类是用来记录被挂起的事务信息的，在事务被挂起之前，会将事务的相关信息保存到SuspendedResourcesHolder对象中，便于后续恢复该挂起的事务时，能够恢复挂起之前的事务状态。保存内容包括该事务数据库连接，事务同步器列表，事务名称，是否只读事务，事务隔离级别，是否是活动的事务。

### TransactionDefinition

事务定义接口，类似于BeanDefinition，定义了事务的描述信息，例如@Transaction注解上标注的信息，默认实现类为DefaultTransactionDefinition。其记录了事务的名称，是否只读事务，事务的传播方式，事务的隔离级别，事务的超时时间。

### TransactionAttribute

TransactionAttribute接口扩展了TransactionDefinition接口，增加了事务管理器名称，事务是否回滚判断接口。默认实现类是DefaultTransactionAttribute，其继承了DefaultTransactionDefinition。

### RuleBasedTransactionAttribute

该类继承了DefaultTransactionAttribute，扩展了回滚规则的设置，支持配置符合指定回滚规则才进行回滚。

### TransactionStatus

该接口定义了事务的状态接口信息，其继承了SavepointManager接口。接口定义了是否是一个新事物，是否仅回滚事务，事务是否已完成，创建保存点，回滚到保存点，释放保存点，是否存在保存点等接口。其默认实现类为DefaultTransactionStatus，其中保存了TransactionDefinition，DataSourceTransactionManager.DataSourceTransactionObject，SuspendedResourcesHolder(如果存在的话)对象。

### TransactionInfo

事务信息类。该类属于保存事务的完整信息的一个类，保存了PlatformTransactionManager，TransactionAttribute，TransactionStatus对象，并提供了将事务信息绑定到当前线程的方法bindToThread()。

### TransactionSynchronizationManager

事务同步管理器。该类内部使用多个ThreadLocal用于保存事务信息，解决多线程事务同步问题，其中包括当前线程数据库连接资源，当前线程事务同步器，当前线程事务名称，当前线程事务隔离级别等。

### DataSourceUtils

该工具类封装了数据库连接和事务之间的一些相关方法，例如获取数据库连接并绑定连接资源到事务同步管理器等。该类中的许多方法是通过TransactionSynchronizationManager类去完成。

### TransactionAttributeSource

该接口只有一个方法getTransactionAttribute(), 就是用来获取事务属性对象。例如@Transaction注解使用AnnotationTransactionAttributeSource获取事务属性信息，其中使用SpringTransactionAnnotationParser解析@Transaction注解。

### TransactionInterceptor

事务拦截器，事务实现的入口，该拦截器会在需要启用事务的方法调用前开启事务，方法调用后提交事务，异常时回滚事务。

### BeanFactoryTransactionAttributeSourceAdvisor

该类是一个Advisor，通过它我们可以将TransactionInterceptor注册为一个通知，其持有TransactionInterceptor和TransactionAttribute。不明白原理的可以看看之前的文章[Spring系列-Spring Aop实现原理分析](https://www.tengjiang.site/spring%E7%B3%BB%E5%88%97/2022/03/16/Spring%E7%B3%BB%E5%88%97-Spring-Aop%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90.html)。

### @TransactionalEventListener

事务事件监听器。该监听器可以通过phase属性配置监听的事务的阶段，在指定的阶段进行指定事件的通知处理。

## 各类的引用关系

```yaml
TransactionInfo
 - PlatformTransactionManager
 - TransactionAttribute
 - TransactionStatus
  - TransactionDefinition
  - SuspendedResourcesHolder
   - old ConnectionHolder + config
  - DataSourceTransactionManager.DataSourceTransactionObject
   - ConnectionHolder
    - Connection
```