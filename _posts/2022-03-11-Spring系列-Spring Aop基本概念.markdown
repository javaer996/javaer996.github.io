---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring Aop基本概念
subtitle: 要先把手放开，才抓得住精彩旳未来。
date:   2022-03-11 16:39:20 +0800
description: Spring系列-Spring Aop基本概念 AspectJ 动态代理
categories: [Spring系列]
tags: [spring]
author: tengjiang
toc: yes
---


在我们讲解Spring Aop之前，我们先要说一下Aop。Aop是面向对象的一种补充，可以让我们以面向切面的方式进行编程。面向切面的编程是一种编程范式，旨在通过允许横切关注点的分离，提高模块化。

Aop实现的关键就是把切面应用到目标对象，实现Aop的技术可分为静态织入和动态代理。

- 静态织入又可分为编译时织入、编译后织入和类加载时织入；
- 动态代理则是在运行时创建代理对象。

## Spring Aop和AspectJ有什么区别

| Spring AOP                                       | AspectJ                                                      |
| ------------------------------------------------ | ------------------------------------------------------------ |
| 由Spring使用动态代理方式实现                     | 是一个完整的Aop解决方案，比Spring Aop更强大                  |
| 不需要单独的编译过程                             | 除非设置 LTW，否则需要 AspectJ 编译器 (ajc)                  |
| 仅支持运行时织入                                 | 支持编译时、编译后和加载时织入                               |
| 仅支持方法级编织                                 | 可以编织字段、方法、构造函数、静态初始值设定项、最终类/方法等 |
| 只能在由 Spring 容器管理的 bean 上实现           | 可以在所有域对象上实现                                       |
| 仅支持方法执行切入点                             | 支持所有切入点                                               |
| 代理是由目标对象创建的, 并且切面应用在这些代理上 | 在执行应用程序之前直接在代码中进行织入                       |
| 性能较低                                         | 性能高                                                       |

总而言之，Spring Aop是Aop的一种解决方案，它不是一个完整的Aop实现，而是在 AOP 实现和 Spring IoC 之间提供紧密的集成，以帮助解决企业应用程序中的常见问题。AspectJ是一个与Spring无关的Aop实现。可能有的同学有疑问了，为什么Spring中也有AspectJ呢？就像我们经常在Spring使用的@AspectJ注解！

这里说明下，并不是说Spring中直接使用了AspectJ，Spring 使用 AspectJ 提供的用于切入点解析和匹配的库来解释AspectJ 5相同的注解。但是，Spring AOP 运行时仍然是纯 Spring AOP，是使用动态代理实现的，并且不依赖于 AspectJ 编译器或编织器。

## Aop基本概念

### 通知(Advice)

通知定义了在切入点代码执行时间点附近需要做的工作。

Spring支持五种类型的通知：

- Before：在方法调用之前执行，详情查看MethodBeforeAdviceInterceptor
- After-returning：在方法返回之后执行，详情查看AfterReturningAdviceInterceptor
- After-throwing： 在方法抛出异常后执行，不抛异常不执行，详情查看AspectJAfterThrowingAdvice
- Around：在方法执行前后执行，Around的优先级比Before高，详情查看AspectJAroundAdvice
- Introduction： 引介增强，上面的四种类型都是增强方法，引介增强是增强类，可以在运行时动态的添加某些方法。要想动态添加方法，需要定义接口，一般该方法定义的接口实现类要实现DelegatingIntroductionInterceptor类

> 注意：
>
> MethodBeforeAdviceInterceptor是由AspectJMethodBeforeAdvice适配出来的，因为AspectJMethodBeforeAdvice没有实现MethodIntercetor接口。
>
> AfterReturningAdviceInterceptor是由AspectJAfterReturningAdvice适配出来的，因为AspectJAfterReturningAdvice没有实现MethodIntercetor接口。
>
> 而AspectJAfterThrowingAdvice，AspectJAroundAdvice本身就实现了MethodIntercetor接口，所以是没有经过适配的。
>
> 这里不知道为什么同样是Advice，有的实现了MethodIntercetor接口，有的没有实现，而是通过适配器去转换，区别对待？

### 连接点(Joinpoint)

程序能够应用通知的一个“时机”，这些“时机”就是连接点，例如方法调用前、异常抛出时、方法返回后等。这个概念已经隐含在通知Advice中了。

### 切入点(Pointcut)

通知定义了切面要发生的“故事”，连接点定义了“故事”发生的时机，那么切入点就定义了“故事”发生的地点，例如某个类或方法的名称，Spring中允许我们方便的用正则表达式来指定。

切入点表达式有10种：

- execution：用于匹配方法执行的连接点
- within：用于匹配指定类型内的方法执行
- this：用于匹配当前AOP代理对象类型的执行方法；注意是AOP代理对象的类型匹配，这样就可能包括引入接口也* 类型匹配
- target：用于匹配当前目标对象类型的执行方法；注意是目标对象的类型匹配，这样就不包括引入接口也类型匹配
- args：用于匹配当前执行的方法传入的参数为指定类型的执行方法
- @within：用于匹配所以持有指定注解类型内的方法
- @target：用于匹配当前目标对象类型的执行方法，其中目标对象持有指定的注解
- @args：用于匹配当前执行的方法传入的参数持有指定注解的执行
- @annotation：用于匹配当前执行方法持有指定注解的方法
- bean：Spring AOP扩展的，AspectJ没有对于指示符，用于匹配特定名称的Bean对象的执行方法

execution格式：

```xml
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throws-pattern?)
```

- 其中带 ?号的 modifiers-pattern?，declaring-type-pattern?，hrows-pattern?是可选项
- ret-type-pattern,name-pattern, parameters-pattern是必选项
- modifier-pattern? 修饰符匹配，如public 表示匹配公有方法
- ret-type-pattern 返回值匹配，* 表示任何返回值，全路径的类名等
- declaring-type-pattern? 类路径匹配
- name-pattern 方法名匹配，* 代表所有，set*，代表以set开头的所有方法
- (param-pattern) 参数匹配，指定方法参数(声明的类型)，(..)代表所有参数，(,String)代表第一个参数为任何值,第 * 二个为String类型，(..,String)代表最后一个参数是String类型
- throws-pattern? 异常类型匹配

### 切面(Aspect)

通知、连接点、切入点共同组成了切面：时间、地点和要发生的“故事”。

### 引入(Introduction)

引入允许我们向现有的类添加新的方法和属性。

### 目标(Target)

即被通知的对象。

### 代理(proxy)

应用通知的对象。

### 织入(Weaving)

把切面应用到目标对象来创建新的代理对象的过程，织入一般发生在如下几个时机:

- 编译时：编译时进行织入，这需要特殊的编译器才可以做的到，例如AspectJ的织入编译器；

- 类加载时：使用特殊的ClassLoader在目标类被加载到程序之前增强类的字节代码；

- 运行时：切面在运行的某个时刻被织入，SpringAOP就是以这种方式织入切面的，原理是动态代理技术。

## Spring Aop的基本概念

### Advisor

封装了通知(Advice)和切入点(Pointcut)的对象称为Advisor。

### MethodInterceptor

顾名思义，方法拦截器，该接口继承了Advice接口，定义了invoke()方法。Spring Aop所有的切面通知最种都要转换成方法拦截器才能去执行。

### IntroductionInterceptor

引介增强接口，如果要实现引介增强，增强接口的实现类必须实现IntroductionInterceptor接口。

### 动态代理

#### JDK动态代理

JDK动态代理要求被代理类必须实现了接口。如果要增强的方法实现类实现了接口，默认会使用JDK动态代理，其原理是通过接口实现代理类。

#### CGLIB动态代理

如果被代理类没有实现接口，会使用CGLIB动态代理，其原理是通过生成该类的子类进行代理。如果proxyTargetClass为true，则强制走CGLIB动态代理。