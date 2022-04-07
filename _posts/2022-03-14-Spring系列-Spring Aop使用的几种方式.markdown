---
layout: post
read_time: true
show_date: true
title:  Spring系列-Spring Aop使用的几种方式
subtitle: 戏言不能伤敌，但能伤友。
date:   2022-03-14 16:39:20 +0800
description: Spring系列-Spring Aop使用的几种方式
categories: [Spring]
tags: [spring, Spring系列]
author: tengjiang
toc: yes
---

Spring中有多种方式可以使用Spring Aop，例如：

- 基于ProxyFactoryBean

- 基于BeanNameAutoProxyCreator或DefaultAdvisorAutoProxyCreator

- 基于AspectJAwareAdvisorAutoProxyCreator

- 基于AnnotationAwareAspectJAutoProxyCreator

下面我们分别介绍以下这几种方式都是怎么使用的。

## 基于ProxyFactoryBean

### xml配置方式

```java
// 定义接口
public interface UserService {
  String getName();
}

// 定义实现类
public class UserService1 implements UserService {
  @Override
  public String getName() {
    System.out.println("方法执行");
    return "UserService1";
  }
}

// 定义拦截器
public class AroundInterceptor implements MethodInterceptor {
  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    System.out.println("方法执行之前");
    Object proceed = invocation.proceed();
    System.out.println("方法执行之后");
    return proceed;
  }
}

// 启动
public static void main( String[] args ) {
  ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:spring.xml");
  UserService userService = (UserService) applicationContext.getBean("userService");
  String name = userService.getName();
  System.out.println(name);
}
```

xml配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">
  <bean id="userService1" class="com.demo.aop.base.UserService1"/>
  <bean id="aroundInterceptor" class="com.demo.aop.base.AroundInterceptor"/>
  <!-- 设置拦截器 -->
  <bean id="userService" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target" ref="userService1"/>
    <property name="interceptorNames" value="aroundInterceptor"/>
  </bean>
</beans>
```

### 注解配置方式

```java
// 定义接口
public interface UserService {
  String getName();
}

// 定义实现类，添加@Component注解
@Component
public class UserService1 implements UserService {
  @Override
  public String getName() {
    System.out.println("方法执行");
    return "UserService1";
  }
}

// 定义拦截器，添加@Component注解
@Component
public class AroundInterceptor implements MethodInterceptor {
  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    System.out.println("方法执行之前");
    Object proceed = invocation.proceed();
    System.out.println("方法执行之后");
    return proceed;
  }
}

// 定义配置类
@Configuration
public class InterceptorConfig {

  // 配置代理FactoryBean
  @Bean
  public ProxyFactoryBean userService(UserService1 userService1) {
    ProxyFactoryBean proxyFactoryBean = new ProxyFactoryBean();
    proxyFactoryBean.setTarget(userService1);
    proxyFactoryBean.setInterceptorNames("aroundInterceptor");
    return proxyFactoryBean;
  }
}

// 启动
public static void main( String[] args ) {
  AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext("com.demo.aop.base.anno");
  UserService userService = (UserService) applicationContext.getBean("userService");
  String name = userService.getName();
  System.out.println(name);
}
```

## 基于DefaultAdvisorAutoProxyCreator

使用DefaultAdvisorAutoProxyCreator的时候，因为其只识别Advisor类型，不识别Advice和MethodInterceptor类型，所以我们需要将Advice或MethodInterceptor封装到Advisor才可正常使用。

> BeanNameAutoProxyCreator类似，就不做介绍了。

### xml配置方式

```java
// 定义接口
public interface UserService {
  String getName();
}

// 定义实现类
public class UserService1 implements UserService {
  @Override
  public String getName() {
    System.out.println("方法执行");
    return "UserService1";
  }
}

// 定义拦截器
public class AroundInterceptor implements MethodInterceptor {
  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    System.out.println("方法执行之前");
    Object proceed = invocation.proceed();
    System.out.println("方法执行之后");
    return proceed;
  }
}

// 启动
public static void main( String[] args ) {
  ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop/autoproxy/spring.xml");
  UserService userService = (UserService) applicationContext.getBean("userService1");
  String name = userService.getName();
  System.out.println(name);
}
```

xml配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">
  
  <bean id="userService1" class="com.demo.aop.autoproxy.UserService1"/>
  <bean id="aroundInterceptor" class="com.demo.aop.autoproxy.AroundInterceptor"/>
  <!-- 设置自动代理类 -->
  <bean id="defaultAdvisorAutoProxyCreator" class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>
  <!-- 设置pointcut，拦截所有get开头的方法 -->
  <bean id="pointcut" class="org.springframework.aop.support.NameMatchMethodPointcut">
    <property name="mappedName" value="get*"/>
  </bean>
  <!-- 设置advisor,封装advice和pointcut -->
  <bean id="advisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
    <property name="advice" ref="aroundInterceptor"/>
    <property name="pointcut" ref="pointcut"/>
  </bean>
</beans>
```

### 注解配置方式

```java
// 定义接口
public interface UserService {
  String getName();
}

// 定义实现类，添加@Component注解
@Component
public class UserService1 implements UserService {
  @Override
  public String getName() {
    System.out.println("方法执行");
    return "UserService1";
  }
}

// 定义拦截器，添加@Component注解
@Component
public class AroundInterceptor implements MethodInterceptor {
  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    System.out.println("方法执行之前");
    Object proceed = invocation.proceed();
    System.out.println("方法执行之后");
    return proceed;
  }
}

// 定义配置类
@Configuration
public class InterceptorConfig {
  
  // 定义自动配置类
  @Bean
  public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
    return new DefaultAdvisorAutoProxyCreator();
  }

  // 定义pointcut
  // 这里只是随便挑了一个pointcut的实现类
  @Bean
  public Pointcut pointcut() {
    NameMatchMethodPointcut nameMatchMethodPointcut = new NameMatchMethodPointcut();
    nameMatchMethodPointcut.setMappedName("get*");
    return nameMatchMethodPointcut;
  }

  // 定义advisor，封装MethodInterceptor
  @Bean
  public DefaultPointcutAdvisor defaultPointcutAdvisor(AroundInterceptor aroundInterceptor) {
    DefaultPointcutAdvisor defaultPointcutAdvisor = new DefaultPointcutAdvisor();
    defaultPointcutAdvisor.setPointcut(pointcut());
    defaultPointcutAdvisor.setAdvice(aroundInterceptor);
    return defaultPointcutAdvisor;
  }
}

// 启动
public static void main( String[] args ) {
  AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext("com.demo.aop.autoproxy.anno");
  UserService userService = (UserService) applicationContext.getBean("userService1");
  String name = userService.getName();
  System.out.println(name);
}
```

## 基于AspectJAwareAdvisorAutoProxyCreator

该实现类是AspectJ切面写法的Xml配置方式实现类。

```java
// 定义接口
public interface UserService {
  String getName();
}

// 定义实现类
public class UserService1 implements UserService {
  @Override
  public String getName() {
    System.out.println("方法执行");
    return "UserService1";
  }
}

// 定义advice
public class MyBeforeAdvice {
  public void before() {
    System.out.println("方法执行之前");
  }
}

// 启动
public static void main( String[] args ) {
  ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:aop/aspect/spring.xml");
  UserService userService = (UserService) applicationContext.getBean("userService1");
  String name = userService.getName();
  System.out.println(name);
}
```

xml配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
                           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">
  <bean id="userService1" class="com.demo.aop.aspect.UserService1"/>
  <bean id="myBeforeAdvice" class="com.demo.aop.aspect.MyBeforeAdvice"/>
  <!-- 使用aop命名空间配置，expose-proxy为true, 可以通过AopUtils获取代理类，proxy-target-class为true会强制使用CGLIB代理 -->
  <aop:config expose-proxy="true" proxy-target-class="true">
    <aop:pointcut id="pointcut" expression="execution (* com.demo.aop.aspect.UserService1.getName())"/>
    <aop:aspect ref="myBeforeAdvice">
      <aop:before method="before" pointcut-ref="pointcut"/>
    </aop:aspect>
  </aop:config>

</beans>
```

## 基于AnnotationAwareAspectJAutoProxyCreator

该实现类继承了AspectJAwareAdvisorAutoProxyCreator，并扩展了注解方式。

```java
// 定义接口
public interface UserService {
  String getName();
}

// 定义实现类，添加@Component注解
@Component
public class UserService1 implements UserService {
  @Override
  public String getName() {
    System.out.println("方法执行");
    return "UserService1";
  }
}

// 定义AspectJ
@Component
// 启用AspectJ自动代理，对应xml配置中的<aop:aspectj-autoproxy/>
@EnableAspectJAutoProxy
// 定义AspectJ
@Aspect
public class AroundAspect {

  // 定义pointcut
  @Pointcut("execution(* com.demo.aop.aspect.anno.UserService1.getName())")
  public void pointcut(){}

  // 定义advice
  @Around(value = "pointcut()")
  public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
    System.out.println("执行前");
    Object proceed = joinPoint.proceed();
    System.out.println("执行后");
    return proceed;
  }
}

// 启动
public static void main( String[] args ) {
  AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(App.class);
  UserService userService = (UserService) applicationContext.getBean("userService1");
  String name = userService.getName();
  System.out.println(name);
}
```

