---
layout: post
read_time: true
show_date: true
title:  k8s部署Skywalking(简易版)
subtitle: 后悔过去，不如奋斗将来。
date:   2022-02-01 08:58:20 +0800
description: k8s(kubernetes)部署Skywalking(简易版)
categories: [教程]
tags: [k8s,Skywalking]
author: tengjiang
toc: yes
---

### 一、简单介绍

如果要部署skywalking需要部署以下几个服务(选用es7+nas存储)：

**1. nas存储**

**2. es7**

**3. skywalking**

**4. skywalking-ui**

**5. 要监控的服务**

> 注意：部署都是基于同一个命名空间，并且没有进行权限控制，下面某些镜像的打包如果觉得没必要可以直接用原始的镜像，不必非打包自己的镜像。

### 二、开始部署

#### 1、创建nas，并创建对应的pv，pvc(从阿里控制台操作)

详情请参考阿里云文档

#### 2、编辑ES7的k8s配置文件 (这里使用的单节点)

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es7
  namespace: dev
spec:
  replicas: 1
  serviceName: es7
  selector:
    matchLabels:
      app: es7
  template:
    metadata:
      labels:
        app: es7
    spec:
      nodeSelector:
        env: dev
      containers:
        - name: es7
          image: docker.elastic.co/elasticsearch/elasticsearch:7.6.1
          ports:
            - containerPort: 9200
              name: rest
              protocol: TCP
            - containerPort: 9300
              name: inter-node
              protocol: TCP
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
          env:
            - name: path.data
              value: /usr/share/elasticsearch/data
            - name: discovery.type
              value: "single-node"
            - name: bootstrap.memory_lock
              value: "false"
            - name: bootstrap.system_call_filter
              value: "false"
            - name: cluster.name
              value: es-cluster
#            - name: discovery.zen.minimum_master_nodes # 含义请参阅官方 Elasticsearch 文档
#              value: "1"
#            - name: discovery.seed_hosts # 含义请参阅官方 Elasticsearch 文档
#              value: "annoroad-es7-service-internal"
#            - name: cluster.initial_master_nodes # 初始化的 mastes7-cluster-0,es7-cluster-1,es7-cluster-2" # 含义请参阅官方 Elasticsearch 文档
#              valueFrom:
#                fieldRef:
#                  fieldPath: metadata.name
#            - name: node.name
#              valueFrom:
#                fieldRef:
#                  fieldPath: metadata.name
            - name: indices.breaker.total.limit  # 避免发生OOM
              value: "60%"
            - name: indices.fielddata.cache.size  # 有了这个设置，最久未使用（LRU）的 fielddata 会被回收为新数据腾出空间
              value: "40%"
            - name: indices.breaker.fielddata.limit  # fielddata 断路器默认设置堆的  作为 fielddata 大小的上限
              value: "60%"
            - name: indices.breaker.request.limit  # request 断路器估算需要完成其他请求部分的结构大小，例如创建一个聚合桶，默认限制是堆内存
              value: "60%"
            # 父级断路器是否应考虑实际内存使用情况（true）或仅考虑子级断路器保留的数量（false）。默认为true
            # 整个父级断点器的启动限制，如果indices.breaker.total.use_real_memory: false 默认为JVM堆的70％ of the JVM heap., 否则为95% of the JVM heap
            - name: indices.breaker.total.use_real_memory
              value: "false"
            - name: ES_JAVA_OPTS
              value: "-Xms2g -Xmx2g" # 根据具体资源及需求调整
      initContainers:
        - name: increase-vm-max-map
          image: busybox
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          securityContext:
            privileged: true
        - name: increase-fd-ulimit
          image: busybox
          command: ["sh", "-c", "ulimit -n 65536"]
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: es7-nas-pvc
```

#### 3、编辑ES7的Service的k8s配置文件

```yaml
apiVersion: v1
kind: Service
metadata:
  name: es7-service-internal
  namespace: dev
  labels:
    app: es7-service-internal
spec:
  ports:
  - port: 9200
    name: rest
    protocol: TCP
    targetPort: 9200
  - port: 9300
    name: inter-node
    protocol: TCP
    targetPort: 9300
  selector:
    app: es7
  clusterIP: None
```

#### 4、编辑Skywalking的Dockerfile文件

```yaml
# 指定操作的镜像，不指定版本默认使用最新版本
#FROM
FROM apache/skywalking-oap-server:8.1.0-es7

# 维护者信息
MAINTAINER xxx <xxx@xxx.com>

ENV TZ=Asia/Shanghai

RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime
RUN echo $TZ > /etc/timezone
```
这里之所以使用Dockerfile重新打了镜像，是因为一开始下载了skywalking的所有jar包和启动文件，想要依据于代码打成我们自己的镜像，如果想要更改什么配置，只需要在skywalking项目中修改配置后，使用gitlab直接重新打包发布，和普通项目一样的操作，但是后来测试中一直报OOM的错误，本地没问题，所以后来改为了直接依赖于基础的skywalking镜像进行镜像的重新创建发布

#### 5、编辑Skywalking的k8s的配置文件

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: skywalking
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: skywalking
  template:
    metadata:
      labels:
        app: skywalking
    spec:
      nodeSelector:
        env: dev
      containers:
        - name: skywalking
          # 这里使用的是上面自己生成的镜像
          image: registry-vpc.cn-beijing.aliyuncs.com/cloud/skywalking_dev:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 11800
              name: grpc
            - containerPort: 12800
              name: rest
          env:
            - name: JAVA_OPTS
              value: "-Xmx512m -Xms512m"
            - name: SW_CLUSTER
              value: standalone
            - name: SW_STORAGE
              value: elasticsearch7
            - name: SW_STORAGE_ES_CLUSTER_NODES
              value: es7-service-internal:9200
            - name: SW_NAMESPACE
              value: dev
            - name: SKYWALKING_COLLECTOR_UID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.uid
```

#### 6、编辑SkyWalking的Service的k8s的启动配置文件

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
  name: skywalking-service-internal
  namespace: dev
  labels:
    app: skywalking-service-internal
spec:
  ports:
  - port: 11800
    name: grpc-port
    protocol: TCP
    targetPort: 11800
  - port: 12800
    name: http-port
    protocol: TCP
    targetPort: 12800
  selector:
    app: skywalking
  type: ClusterIP
```

#### 7、编辑SkyWalking-ui的Dockerfile文件(原因同上)

```yaml
# 指定操作的镜像，不指定版本默认使用最新版本
#FROM
FROM apache/skywalking-ui:8.1.0

# 维护者信息
MAINTAINER xxx <xxx@xxx.com>

ENV TZ=Asia/Shanghai

RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime
RUN echo $TZ > /etc/timezone
```

#### 8、编辑SkyWalking-ui的k8s的配置文件

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: skywalking-ui
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: skywalking-ui
  template:
    metadata:
      labels:
        app: skywalking-ui
    spec:
      nodeSelector:
        env: dev
      containers:
        - name: skywalking-ui
          image: registry-vpc.cn-beijing.aliyuncs.com/xxx/skywalking-ui_dev:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: SW_OAP_ADDRESS
              value: skywalking-service-internal:12800
```

#### 9、编辑SkyWalking-ui的Service的k8s的配置文件

```yaml
# 这里我们使用的是LoadBalancer类型的Service，因为我们是部署在阿里云上，依赖于阿里云的支持，否则的话
# 可以使用NodePort的方式进行端口的暴露，进行访问
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/alicloud-loadbalancer-address-type: "xxx"
    service.beta.kubernetes.io/alicloud-loadbalancer-id: "xxx"
    service.beta.kubernetes.io/alicloud-loadbalancer-force-override-listeners: "true"
  name: skywalking-ui-service-internal
  namespace: dev
  labels:
    app: skywalking-ui-service-internal
spec:
  ports:
  - port: 12800
    protocol: TCP
    targetPort: 8080
  selector:
    app: skywalking-ui
  type: LoadBalancer
```

#### 10、编辑skywalking-agent的配置文件

```yaml
# 指定操作的镜像，不指定版本默认使用最新版本
FROM busybox:latest

# 维护者信息
MAINTAINER xxx <xxx@xxx.com>

ENV LANG=C.UTF-8

RUN set -eux && mkdir -p /usr/skywalking/agent/

ADD /agent/ /usr/skywalking/agent/

WORKDIR /
```

这里是下载了skywalking的zip包文件，然后将agent目录下的文件打包到了镜像中。

agent的打包需要注意，plugins目录下的插件才会被使用，optional-plugins目录下的插件并不会被使用。

因为我们监控的使用可能会有很多我们不想监控的心跳等信息，如果需要忽略某些信息，需要将optional-plugins目录下的 apm-trace-ignore-plugin-x.x.x.jar复制到plugins目录下。

如果想要监控springcloud 的gateway，需要将optional-plugins下的apm-spring-cloud-gateway-x.x.x-plugin-x.x.x.jar复制到plugins目录下。

所以这里我直接打了不同的agent镜像，一个用于通用项目监控，镜像名以-ignore后缀结尾，一个用于监控gateway项目，镜像名以-gateway结尾，之所以区分不同镜像，因为默认插件太多了，像gateway项目完全用不到，为了精简镜像所以进行了区分。如果有洁癖的话，还可以进一步区分。

- 下载地址：[https://github.com/apache/skywalking/releases](https://github.com/apache/skywalking/releases)

#### 11、配置需要监控的项目(以sidecar 模式挂载 agent)

```yaml
# 配置项目的Dockerfile
FROM openjdk:8-jdk-alpine

# 维护者信息
MAINTAINER xxx <xxx@xxx.com>

# VOLUME 指向了一个/tmp的目录，由于 Spring Boot 使用内置的Tomcat容器，Tomcat 默认使用/tmp作为工作目录。这个命令的效果是：在>宿主机的/var/lib/docker目录下创建一个临时文件并>把它链接到容器中的/tmp目录
VOLUME /tmp

ARG server_name
ARG server_port

ENV TZ=Asia/Shanghai
ENV SERVER_PORT=$server_port
ENV SERVER_NAME=$server_name

ADD $server_name.jar /server.jar

RUN sh -c 'touch /server.jar' && ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

EXPOSE $server_port

# 为了缩短 Tomcat 的启动时间，添加java.security.egd的系统属性指向/dev/urandom作为 ENTRYPOINT
# 主要是这里，需要配置上skywalking的服务地址，这里我们依赖于k8s的dns
ENTRYPOINT exec java -javaagent:/skywalking/agent/skywalking-agent.jar \
            -Dskywalking.agent.service_name=$SERVER_NAME \
            -Dskywalking.collector.backend_service=skywalking-service-internal:11800 \
            -Djava.security.egd=file:/dev/./urandom \
            -jar /server.jar
```

```yaml
# 配置项目的k8s的配置文件
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: server
  namespace: dev
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: server
  template:
    metadata:
      labels:
        app: server
    spec:
      terminationGracePeriodSeconds: 60
      nodeSelector:
        env: dev
      # 通过initContainers，在容器启动前，将agent的文件挂载到指定的目录中，这样就不用在每一个项目中都拷贝一份agent的jar和配置了,直接通过sidecar的模式使用
      initContainers:
        - image: registry-vpc.cn-beijing.aliyuncs.com/xxx/sw-agent-ignore:latest
          name: sw-agent-sidecar
          imagePullPolicy: Always
          command: ['sh']
          args: ['-c','mkdir -p /skywalking/agent && cp -r /usr/skywalking/agent/* /skywalking/agent']
          volumeMounts:
            - mountPath: /skywalking/agent
              name: sw-agent
      containers:
      - name: server
        env:
        - name: DEPLOY_TAG
          value: tag10
          # 需要忽略的监控请求
        - name: SW_AGENT_TRACE_IGNORE_PATH
          value: /actuator/**,/eureka/**
        volumeMounts:
          - name: sw-agent
            mountPath: /skywalking/agent
        image: registry-vpc.cn-beijing.aliyuncs.com/cloud/server:latest
        imagePullPolicy: Always #每次都重新拉取镜像
        readinessProbe:
          httpGet:
            path: /actuator/info
            port: 30003
          initialDelaySeconds: 30
          timeoutSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/info
            port: 30003
          initialDelaySeconds: 60
          timeoutSeconds: 10
        ports:
        - name: http
          containerPort: 30003
      volumes:
        - name: sw-agent
          emptyDir: {}
```

 至此，我们的项目已经可以被skywalking进行监控了，包括调用链路，服务相关信息。