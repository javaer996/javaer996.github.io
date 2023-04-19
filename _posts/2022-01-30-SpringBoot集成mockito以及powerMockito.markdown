---
layout: post
read_time: true
show_date: true
title:  SpringBoot集成mockito以及powerMockito
subtitle: 
date:   2022-01-30 14:22:20 +0800
description: SpringBoot集成mockito以及powerMockito.
categories: [SpringBoot]
tags: [springboot,mockito]
author: tengjiang
toc: yes
---

### 什么是Mock？
在单元测试中，我们往往想去独立地去测一个类中的某个方法，但是这个类可不是独立的，它会去调用一些其它类的方法和service，这也就导致了以下两个问题：

外部服务可能无法在单元测试的环境中正常工作，因为它们可能需要访问数据库或者使用一些其它的外部系统。
我们的测试关注点在于这个类的实现上，外部类的一些行为可能会影响到我们对本类的测试，那也就失去了我们进行单测的意义。

为了解决这种问题，Mockito和PowerMock诞生了。这两种工具都可以用一个虚拟的对象来替换那些外部的、不容易构造的对象，便于我们进行单元测试。两者的
不同点在于Mockito对于大多数的标准单测case都很适用，而PowerMock可以去解决一些更难的问题（比如静态方法、私有方法等）。

### 使用示例

#### 一、引入依赖

```xml
<!-- 默认会引入mockito-core -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
<!-- 如果需要mock静态方法，私有方法，final方法引入powerMockito -->
<dependency>
  <groupId>org.powermock</groupId>
  <artifactId>powermock-module-junit4</artifactId>
  <version>2.0.0-RC.3</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.powermock</groupId>
  <artifactId>powermock-api-mockito2</artifactId>
  <version>2.0.0-RC.3</version>
  <scope>test</scope>
</dependency>
```

#### 二、测试类特殊示例

```java
@RunWith(PowerMockRunner.class)     // 使用powerMockito
@PrepareForTest(TestService.class)  // 对于需要使用powerMockito的类进行配置
public class RefundServiceTestCase {

    @InjectMocks  // 会将mock的类注入到该类
    private TestService testService;
    @Mock  // mock该类，会拦截所有方法调用, 使用when(obj.do()).doReturn();
    private Test2Service test2Service;
    @Spy   // mock该类，部分mock，没有mock的方法直接调用真实方法，最好使用doReturn()..when(obj).do(), 如果使用when(obj.do()).doReturn()仍然会调用真实方法，只是最后会返回mock的结果
    private Test test;

    @Test
    public void test() throws Exception {

        // mock接口配置
        // 如果需要mock私有方法，必须这样写一下
        testService = PowerMockito.spy(testService);
				
        // mock私有方法
        PowerMockito.doReturn(null)
                .when(testService, "test", any(), any());

        when(test2Service.test2(anyString(), eq(1)))
                .thenReturn(testObject);

        // 调用mock接口
        TestObject testObject = testService.test("test", 1, 2);
        Assert.assertNotNull(testObject);
        Test2Object test2Ojbect = testService2.test2("test2", 1);
        Assert.assertNotNull(test2Object);
    }
```