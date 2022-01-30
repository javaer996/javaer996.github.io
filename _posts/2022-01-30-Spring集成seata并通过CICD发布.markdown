---
layout: post
read_time: true
show_date: true
title:  Spring集成seata并通过CICD发布
date:   2022-01-30 17:40:20 +0800
description: Spring集成seata并通过CICD发布.
img: posts/20220130/seata.png
tags: [spring,seata,gitlab,CICD]
author: tengjiang
toc: yes
---

## seata-server安装启动

### 下载seata-server安装包

- https://github.com/seata/seata/releases

<!-- more -->
### 配置file.conf和registry.conf等配置文件

```yaml
## file.conf
## transaction log store, only used in seata-server
store {
  ## store mode: file、db、redis
  mode = "db"

  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }

  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    driverClassName = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://xxxx/demo_transaction_database"
    user = "test"
    password = "xxxx"
    minConn = 5
    maxConn = 30
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }

  ## redis store property
  redis {
    host = "127.0.0.1"
    port = "6379"
    password = ""
    database = "0"
    minConn = 1
    maxConn = 10
    queryLimit = 100
  }
}
```

```yaml
## registry.conf
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "file"

  nacos {
    application = "seata-server"
    serverAddr = "127.0.0.1:8848"
    group = "SEATA_GROUP"
    namespace = ""
    cluster = "default"
    username = ""
    password = ""
  }
  eureka {
    serviceUrl = "http://localhost:8761/eureka"
    application = "default"
    weight = "1"
  }
  redis {
    serverAddr = "localhost:6379"
    db = 0
    password = ""
    cluster = "default"
    timeout = 0
  }
  zk {
    cluster = "default"
    serverAddr = "127.0.0.1:2181"
    sessionTimeout = 6000
    connectTimeout = 2000
    username = ""
    password = ""
  }
  consul {
    cluster = "default"
    serverAddr = "127.0.0.1:8500"
  }
  etcd3 {
    cluster = "default"
    serverAddr = "http://localhost:2379"
  }
  sofa {
    serverAddr = "127.0.0.1:9603"
    application = "default"
    region = "DEFAULT_ZONE"
    datacenter = "DefaultDataCenter"
    cluster = "default"
    group = "SEATA_GROUP"
    addressWaitTime = "3000"
  }
  file {
    name = "file.conf"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"

  nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = ""
    group = "SEATA_GROUP"
    username = ""
    password = ""
  }
  consul {
    serverAddr = "127.0.0.1:8500"
  }
  apollo {
    appId = "seata-server"
    apolloMeta = "http://192.168.1.204:8801"
    namespace = "application"
  }
  zk {
    serverAddr = "127.0.0.1:2181"
    sessionTimeout = 6000
    connectTimeout = 2000
    username = ""
    password = ""
  }
  etcd3 {
    serverAddr = "http://localhost:2379"
  }
  file {
    name = "file.conf"
  }
}

```

### 配置gitlab自动化发布

#### Dockerfile

```dockerfile
# 指定操作的镜像，不指定版本默认使用最新版本
FROM openjdk:8-jdk-alpine

# 维护者信息
MAINTAINER tengjiang

ARG server_name
ARG server_port

ENV TZ=Asia/Shanghai
ENV SERVER_PORT=$server_port

ADD bin /seata/bin
ADD conf /seata/conf
ADD lib /seata/lib

RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime
RUN echo $TZ > /etc/timezone

EXPOSE $server_port

ENTRYPOINT ["sh", "/seata/bin/seata-server.sh"]
```

#### .gitlab-ci-dockerbuild.sh

```yaml
#!/bin/sh

server_name=$1
annoroad_registry=$2
annoroad_registry_username=$3
annoroad_registry_password=$4
env=$5

if [ $env = "product" ];then
  # 生产环境
  server_port=8091;
elif [ "$env" = "test" ];then
  # 测试环境
  server_port=8091;
elif [ "$env" = "dev" ];then
  # 开发环境
  server_port=8091;
elif [ "$env" = "" ];then
  echo "ERROR: PROFILE is empty !!!!!!!";
  return 1;
else
  echo "ERROR: invalid PROFILE($env) !!!!!!!";
  return 1;
fi

image_name=$annoroad_registry/${server_name}_${env}

cp ./docker/Dockerfile ./seata/Dockerfile
cp -r ./env/$env/. ./seata/conf
cd ./seata

# 镜像构建
docker build --build-arg server_name=$server_name --build-arg server_port=$server_port -t $server_name .
docker tag $server_name $image_name

# 登录私有registry
docker login --username=$annoroad_registry_username --password=$annoroad_registry_password $annoroad_registry

# 打包推送
docker push $image_name

# 删除本地镜像
docker rmi $server_name
docker rmi $image_name
```

#### .gitlab-ci.yml

```yaml
# 本次构建的阶段：build、deploy
stages:
  - build
  - deploy

# define job variables at job level
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=/root/.m2/repository"
  DOCKER_DRIVER: overlay2
  K8S_DEPLOYMENT_NAME: $CI_PROJECT_NAME
  PROJECT_NAME: $CI_PROJECT_NAME
  ANNOROAD_KUBECTL_IMAGE: registry.cn-beijing.aliyuncs.com/xxxx/annoroad-kubectl:2.0.0

# 镜像构建和打包推送阶段
build:
  stage: build
  tags:
    - annoroad-ms-runner
  only:
    - web
  script:
    - echo "=============== 开始 镜像构建和打包推送 ==============="
    - chmod 755 gitlab-ci-dockerbuild.sh
    - ./gitlab-ci-dockerbuild.sh $PROJECT_NAME $ANNOROAD_REGISTRY $ANNOROAD_REGISTRY_USERNAME $ANNOROAD_REGISTRY_PASSWORD $PROFILE

# 应用部署阶段
deploy:
  stage: deploy
  image:
    name: $ANNOROAD_KUBECTL_IMAGE
    # 覆盖原镜像的entrypoint，要不然会直接退出
    entrypoint: [""]
  tags:
    - annoroad-ms-runner
  only:
    - web
  script:
    - echo "=============== 开始 应用部署 ==============="
    # 修改 $K8S_DEPLOYMENT_NAME 中的 ENV 的 DEPLOY_TAG 属性，使k8s重新发布 $K8S_DEPLOYMENT_NAME
    - kubectl set env deploy/$K8S_DEPLOYMENT_NAME DEPLOY_TAG="$(date)" -n $PROFILE
```

### 数据库脚本配置

```sql
CREATE TABLE `branch_table` (
`branch_id` bigint(20) NOT NULL,
`xid` varchar(128) NOT NULL,
`transaction_id` bigint(20) DEFAULT NULL,
`resource_group_id` varchar(32) DEFAULT NULL,
`resource_id` varchar(256) DEFAULT NULL,
`branch_type` varchar(8) DEFAULT NULL,
`status` tinyint(4) DEFAULT NULL,
`client_id` varchar(64) DEFAULT NULL,
`application_data` varchar(2000) DEFAULT NULL,
`gmt_create` datetime(6) DEFAULT NULL,
`gmt_modified` datetime(6) DEFAULT NULL,
PRIMARY KEY (`branch_id`),
KEY `idx_xid` (`xid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `global_table` (
`xid` varchar(128) NOT NULL,
`transaction_id` bigint(20) DEFAULT NULL,
`status` tinyint(4) NOT NULL,
`application_id` varchar(32) DEFAULT NULL,
`transaction_service_group` varchar(32) DEFAULT NULL,
`transaction_name` varchar(128) DEFAULT NULL,
`timeout` int(11) DEFAULT NULL,
`begin_time` bigint(20) DEFAULT NULL,
`application_data` varchar(2000) DEFAULT NULL,
`gmt_create` datetime DEFAULT NULL,
`gmt_modified` datetime DEFAULT NULL,
PRIMARY KEY (`xid`),
KEY `idx_gmt_modified_status` (`gmt_modified`,`status`),
KEY `idx_transaction_id` (`transaction_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `lock_table` (
`row_key` varchar(128) NOT NULL,
`xid` varchar(96) DEFAULT NULL,
`transaction_id` bigint(20) DEFAULT NULL,
`branch_id` bigint(20) NOT NULL,
`resource_id` varchar(256) DEFAULT NULL,
`table_name` varchar(32) DEFAULT NULL,
`pk` varchar(36) DEFAULT NULL,
`gmt_create` datetime DEFAULT NULL,
`gmt_modified` datetime DEFAULT NULL,
PRIMARY KEY (`row_key`),
KEY `idx_branch_id` (`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```



## client端配置

### POM文件引入配置

#### 方式一

- 不需要连接数据库，只进行服务调用
- 将file.conf和registry.conf的方式改为yaml文件配置的方式
- 开启全局事务

```xml
<dependency>
	<groupId>io.seata</groupId>
	<artifactId>seata-spring-boot-starter</artifactId>
	<version>1.3.0</version>
</dependency>
```

#### 方式二

- 需要连接数据库(数据源会自动代理)
- 需要通过feign调用直接解析全局事务并传播
- 进行事务回滚

```xml
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-alibaba-seata</artifactId>
	<version>2.2.0.RELEAS</version>
</dependency>
<!-- 其实spring-cloud-alibaba-seata中已经引入了seata-spring-boot-starte，但是引入的版本过低，所以这里我们重新引入一个高版本的 -->
<dependency>
	<groupId>io.seata</groupId>
	<artifactId>seata-spring-boot-starter</artifactId>
	<version>1.3.0</version>
</dependency>
```

### 添加配置文件(默认有很多配置项)

> ```yaml
> seata:
>   config:
>     type: file
>   registry:
>     type: file
>   enable-auto-data-source-proxy: true # 数据源自动代理
>   tx-service-group: demo-group  # 事务分组
>   service:
>     vgroup-mapping:
>       annoroad-openapi-group: default
>     grouplist:
>       default: 127.0.0.1:8091 # 该分组事务的server地址
> ```

### 在需要使用分布式事务的库中新建undo_log表

```sql
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```

### 代码中使用方式

#### 开启全局事务

```java
@GlobalTransactional
@Transactional(rollbackFor = Exception.class)
public void enter(){
  // 调用demo方法
  // 调用demo2方法
  // 写库操作
}
```

#### 配置spring事务

```java
@Transactional(rollbackFor = Exception.class)
public void demo(){
  // 写库操作
}
@Transactional(rollbackFor = Exception.class)
public void demo2(){
  // 写库操作
}
```

### 注意事项

- 如果想要分布式事务生效，需要保证全局事务ID的正确传递

  - header中传递(TX_XID)
  - 获取方式(RootContext.getXID())

- 如果项目中使用了@EnableWebMvc或者存在继承了WebMvcConfigurationSupport的配置类，需要手动添加拦截器，原因是不会执行seata中的自动配置类中的添加拦截器方法了

  ```java
  public void addInterceptors(InterceptorRegistry registry) {
              registry.addInterceptor(new SeataHandlerInterceptor()).addPathPatterns(new String[]{"/**"});
  }
  ```