---
layout: post
read_time: true
show_date: true
title:  Skywalking使用方式介绍
subtitle: 再急，也要注意语气^_^
date:   2022-01-31 16:16:20 +0800
description: Skywalking使用方式介绍
categories: [教程]
tags: [Skywalking]
author: tengjiang
toc: yes
---


## Skywalking是什么？

Skywalking是分布式系统的应用程序性能监控工具，特别为微服务、原生云和基于容器的(Kubernetes)架构设计。

Skywalking分为**客户端**和**服务端**两部分。

## 下载Skywalking

### 下载SkyWalking软件包

- [http://skywalking.apache.org/downloads/](http://skywalking.apache.org/downloads/)

> **软件包中包含服务端程序和客户端agent。**

#### 服务端

解压软件包，通过apache-skywalking-apm-bin/bin/startup.sh 启动server端。

默认端口为11800，web可视化服务端口为8080，如果启动成功，浏览器输入localhost:8080会显示监控面板。

#### 客户端

找到client端需要的skywalking-agent.jar,默认在软件包的apache-skywalking-apm-bin/agent/ 目录下。

在需要监控的项目启动时，添加以下参数：

```shell
-javaagent:/Users/xxx/Downloads/apache-skywalking-apm-bin/agent/skywalking-agent.jar
-Dskywalking.agent.service_name=apm-demo1
-Dskywalking.collector.backend_service=localhost:11800
```

## 使用skywalking-agent.jar的方式

### 在普通项目中

直接将相关包拷贝到对应项目中

### 在docker容器中

1. 使用官方基础镜像，只需在构建镜像的时候From这个基础镜像
- apache/skywalking-base
2. 使用dockerfile将agent包构建到已经存在的基础镜像中

### 在K8S中部署

#### 使用sidecar的方式挂载agent

##### 1. 使用dockerfile创建要sidecar挂载的agent

```dockerfile
FROM busybox:latest 
ENV LANG=C.UTF-8
RUN set -eux && mkdir -p /usr/skywalking/agent/
ADD apache-skywalking-apm-bin/agent/ /usr/skywalking/agent/
WORKDIR /
```

##### 2. 在Deployment中使用initContainers中引入agent

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: demo-sw
  name: demo-sw
spec:
  replicas: 1
  selector:
    matchLabels:
      name: demo-sw
  template:
    metadata:
      labels:
        name: demo-sw
    spec:
      initContainers:
      - image: innerpeacez/sw-agent-sidecar:latest
        name: sw-agent-sidecar
        imagePullPolicy: IfNotPresent
        command: ['sh']
        args: ['-c','mkdir -p /skywalking/agent && cp -r /usr/skywalking/agent/* /skywalking/agent']
        volumeMounts:
        - mountPath: /skywalking/agent
          name: sw-agent
      containers:
      - image: nginx:1.7.9
        name: nginx
        volumeMounts:
        - mountPath: /usr/skywalking/agent
          name: sw-agent
        ports:
        - containerPort: 80
      volumes:
      - name: sw-agent
        emptyDir: {}
```

## 扩展插件

Skywalking生态还有一些插件扩展，例如Oracle、Resin插件等。这部分插件主要是由于许可证不兼容/限制，Skywalking无法将这部分插件直接打包到Skywalking安装包内，于是托管在这个地址：[https://github.com/SkyAPM/java-plugin-extensions]( https://github.com/SkyAPM/java-plugin-extensions)