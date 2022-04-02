---
layout: post
read_time: true
show_date: true
title:  SpringBoot配置logback日志示例大全
subtitle: 保持青春的秘诀，是有一颗不安分的心。
date:   2022-02-07 15:35:20 +0800
description: springboot logback
categories: [教程]
tags: [springboot,logback]
author: tengjiang
toc: yes
---

-  springboot默认使用的日志框架是logback；
-  想使用spring扩展profile支持，要以logback-spring.xml命名，其他如property需要改为springProperty

## 一、configuration

### 1. scan
当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true。

### 2. scanPeriod
设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。

### 3. debug
当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。


```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false">

</configuration>
```

## 二、contextName

每个logger都关联到logger上下文，默认上下文名称为“default”。但可以使用contextName标签设置成其他名字，用于区分不同应用程序的记录。

## 三、property

用来定义变量值的标签，property标签有两个属性，name和value；其中name的值是变量的名称，value的值时变量定义的值。通过property定义的值会被插入到logger上下文中。定义变量后，可以使“${name}”来使用变量

```xml
<property name="log.name" value="annoroad-log-demo"/>

<property name="log.pattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS}[%-5level]%logger{80}:%5line-%msg%n" />

<substitutionProperty name="log.base" value="./logs/${log.name}"/>
```

## 四、appender

### 1. 定义
用于定义日志如何输出

### 2. 属性
- name：

   用于定义该appender的名称，该名称需要在root或者logger的配置中使用

- class：

  用于指定实现该appender的实现类

- - ConsoleAppender：

> 把日志添加到控制台

```xml
<appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
  <encoder>
     <pattern>${log.pattern}</pattern>
  </encoder>
</appender>
```

- - FileAppender：

>  把日志添加到文件

- - RollingFileAppender：

> 滚动记录文件，先将日志记录到指定文件，当符合某个条件是，将日志记录到其他文件，是FileAppender的子类

```xml
<!-- 定义appender的名称为logfile，用于在logger后者root中进行引用-->
<appender name="logfile"  class="ch.qos.logback.core.rolling.RollingFileAppender">
    <!-- 指定日志打印级别阈值，只能比logger或者root中设置的高，低了无效，如果logger中level设置的info，这里设置的warn，那么只打印warn级别往上的日志 -->
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
       <level>warn</level>
   </filter>

   <!-- 上面限制的是范围，这里可以针对单个进行处理，如下设置表示，只有info级别的日志才会被打印，其他的都会拒绝打印，当然前提是level或者上面的那个filter的级别设置的低于等于info才有效 -->
   <filter class="ch.qos.logback.classic.filter.LevelFilter">
      <level>info</level>
        <!-- 表示如果是info则通过可以打印-->
      <onMatch>ACCEPT</onMatch>
        <!-- 表示如果不是info则拒绝，不可以打印-->
      <onMismatch>DENY</onMismatch>
   </filter>

   <!-- 设置日志文件的策略 -->
   <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <!--设置文件名的生成规则,以及日志文件存放的路径--> 
        <!-- 这里表示每天生成一个日志文件，我们可以修改成每月生成一个，或者每小时生成一个 -->
        <FileNamePattern>${log.base}/${log.name}.%d{yyyy-MM-dd}.log</FileNamePattern>
        <!-- 最多保留10天的日志文件 -->
        <MaxHistory>10</MaxHistory>
  </rollingPolicy>

 <!-- 这里用来设置日志的打印格式 -->
 <encoder>
    <pattern>${log.pattern}</pattern>
 </encoder>

</appender>
```

## 五、logger

### 1. 定义

   用于指定什么包或者类下的什么类型的日志怎么输出

### 2. 属性
- name：

   用于指定要使用该logger配置的包名或类名(全路径)

- additivity：

   用来描述是否向上级logger传递打印信息。默认是true。

   简单说就是需不需要将要打印的信息也让上级logger打印，如果为true的话，日志就会打印多遍。

```xml
<!-- 指定该logger控制test目录下的日志，并且不让不向上级传递打印信息 -->
<logger name="com.annoroad.log.demo.test" additivity="false">
    <!-- 指定只打印info以上的日志 (TRACE、DEBUG、INFO、WARN、ERROR) -->
   <level value="info"/>
    <!-- 指定appender为demo，可以指定多个appender-->
   <appender-ref ref="demo" />
   <!--<appender-ref ref="logfile" />-->
</logger>
```

## 六、root

   一种特殊的logger，其实也是logger，如果没有单独制定logger，默认就是用root

```xml
<root>
    <level value="INFO"/><!-- TRACE、DEBUG、INFO、WARN和ERROR -->
    <appender-ref ref="stdout"/>
</root>
```

## 七、例子

```xml
<configuration>
	<!-- 定义日志文件名称 -->
	<property name="log.name" value="annoroad-log-demo"/>
	<!-- 定义日志格式 -->
	<property name="log.pattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS}[%-5level]%logger{80}:%5line-%msg%n"/>
	<!-- 定义日志存放目录-->
	<substitutionProperty name="log.base" value="./logs/${log.name}"/>

	<!-- 控制台输出 -->
	<appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>${log.pattern}</pattern>
		</encoder>
	</appender>

	<!-- 控制台输出，限制level等级大于warn的才会输出 -->
	<appender name="stdout2" class="ch.qos.logback.core.ConsoleAppender">
		<filter class="ch.qos.logback.classic.filter.ThresholdFilter">
			<level>warn</level>
		</filter>

		<encoder>
			<pattern>${log.pattern}</pattern>
		</encoder>
	</appender>

	<!-- 输出到文件，一天换一个日志文件，只输出warn级别的日志，其他级别的日志都拒绝输出，日志文件最多存10天 -->
	<appender name="logfile" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<filter class="ch.qos.logback.classic.filter.LevelFilter">
			<level>warn</level>
			<onMatch>ACCEPT</onMatch>
			<onMismatch>DENY</onMismatch>
		</filter>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<FileNamePattern>${log.base}/${log.name}.%d{yyyy-MM-dd}.log</FileNamePattern>
			<MaxHistory>10</MaxHistory>
		</rollingPolicy>
		<encoder>
			<pattern>${log.pattern}</pattern>
		</encoder>
	</appender>

	<!-- 输出到文件，一天换一个日志文件，只输出info级别以上的日志信息，但是不输出warn级别的日志，日志文件最多存10天 -->
	<appender name="logfile2" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<filter class="ch.qos.logback.classic.filter.ThresholdFilter">
			<level>info</level>
		</filter>
		<filter class="ch.qos.logback.classic.filter.LevelFilter">
			<level>warn</level>
			<onMatch>DENY</onMatch>
			<onMismatch>ACCEPT</onMismatch>
		</filter>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<FileNamePattern>${log.base}/${log.name}2.%d{yyyy-MM-dd}.log</FileNamePattern>
			<MaxHistory>10</MaxHistory>
			</rollingPolicy>
		<encoder>
			<pattern>${log.pattern}</pattern>
		</encoder>
	</appender>

	<!-- 
		对于test文件夹下的日志输出：
			trace级别以上的输出到控制台，
			warn级别的日志输出到annoroad-log-demo.日期.log文件，
			info级别以上，不包括warn级别的日志输出到annoroad-log-demo2.日期.log文件，
		并且不向上级logger传递打印信息
	-->
	<logger name="com.annoroad.log.demo.test" additivity="false">
		<level value="trace"/>
		<appender-ref ref="stdout" />
		<appender-ref ref="logfile" />
		<appender-ref ref="logfile2" />
	</logger>

	<!-- 
		对于test2文件夹下的日志，warn级别以上的日志输出的控制台，warn以下日志不输出，并且不向上级logger传递打印信息
	-->
	<logger name="com.annoroad.log.demo.test2" additivity="false">
		<level value="info"/>
		<appender-ref ref="stdout2" />
	</logger>

	<!--
		对于starter下面的日志，debug级别以上的控制台输出，并且输出两遍，因为默认向上级logger传递打印信息，所以root也会进行打印
	-->
	<logger name="com.annoroad.springboot.starter">
		<level value="debug"/>
		<appender-ref ref="stdout"/>
	</logger>

	<!--
		其它没有具体设置的都按此设置进行输出
	-->
	<root>
		<level value="INFO"/>
		<appender-ref ref="stdout"/>
	</root>

</configuration>
```