---
layout: post
read_time: true
show_date: true
title:  本地与alpine容器使用lrzsz进行上传下载教程
subtitle: 愚痴的人，一直想要别人了解他。
date:   2022-02-11 12:58:20 +0800
description: 本地与alpine容器使用lrzsz进行上传下载教程
categories: [容器]
tags: [lrzsz,linux,mac]
author: tengjiang
toc: yes
---

## 什么是lrzsz？
lrzsz是一款在linux里可代替ftp上传和下载的程序。

## 什么是alpine？
Alpine 操作系统是一个面向安全的轻型 Linux 发行版。它不同于通常 Linux 发行版，Alpine 采用了 musl libc 和 busybox 以减小系统的体积和运行时资源消耗，但功能上比 busybox 又完善的多，因此得到开源社区越来越多的青睐。在保持瘦身的同时，Alpine 还提供了自己的包管理工具 apk，可以直接通过 apk 命令直接查询和安装各种软件。

Alpine Docker 镜像也继承了 Alpine Linux 发行版的这些优势。相比于其他 Docker 镜像，它的容量非常小，仅仅只有 5 MB 左右（对比 Ubuntu 系列镜像接近 200 MB），且拥有非常友好的包管理机制。

## 为什么需要lrzsz？
因为在使用docker等容器环境发布服务的时候，我们通常不会将ftp等端口暴露出来，而且我们没有直接登录服务器的权限，也就没法通过挂载的方式在服务器直接拿到我们想要的容器内生成的文件。
此时就可以在容器内安装lrzsz，然后通过lrzsz进行容器内文件的上传下载。

## 使用教程
### 1. 容器内安装lrzsz软件

#### 切换apk软件源(连国外软件源速度太慢)

使用阿里镜像

```sh
sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
```

使用科大镜像

```sh
sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
```

#### 更新apk，安装lrzsz以及相关依赖

```sh
apk update
apk add wget gcc g++ make
wget https://ohse.de/uwe/releases/lrzsz-0.12.20.tar.gz
tar -xf lrzsz-0.12.20.tar.gz
cd lrzsz-0.12.20/
./configure
make; make install
ln -s /usr/local/bin/lrz /usr/local/bin/rz
ln -s /usr/local/bin/lsz /usr/local/bin/sz
```

### 2. 安装支持rz/sz命令的客户端软件

> 本文主要讲解mac下如何使用，windows用户请使用`xshell`等工具。

#### 安装iTerm2

```shell
brew cask install iterm2
```

#### 添加脚本文件

> 将这两个脚本文件放到usr/local/bin/目录下，放到其它目录也可以。

**iterm2-send-zmodem.sh**

```shell
#!/bin/bash
osascript -e 'tell application "iTerm2" to version' > /dev/null 2>&1 && NAME=iTerm2 || NAME=iTerm
if [[ $NAME = "iTerm" ]]; then
	FILE=`osascript -e 'tell application "iTerm" to activate' -e 'tell application "iTerm" to set thefile to choose file with prompt "Choose a file to send"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")"`
else
	FILE=`osascript -e 'tell application "iTerm2" to activate' -e 'tell application "iTerm2" to set thefile to choose file with prompt "Choose a file to send"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")"`
fi
if [[ $FILE = "" ]]; then
	echo Cancelled.
	# Send ZModem cancel
	echo -e \\x18\\x18\\x18\\x18\\x18
	sleep 1
	echo
	echo \# Cancelled transfer
else
	/usr/local/bin/sz "$FILE" --escape --binary --bufsize 4096
	sleep 1
	echo
	echo \# Received $FILE
fi
```

**iterm2-recv-zmodem.sh**

```shell
#!/bin/bash
osascript -e 'tell application "iTerm2" to version' > /dev/null 2>&1 && NAME=iTerm2 || NAME=iTerm
if [[ $NAME = "iTerm" ]]; then
	FILE=$(osascript -e 'tell application "iTerm" to activate' -e 'tell application "iTerm" to set thefile to choose folder with prompt "Choose a folder to place received files in"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")")
else
	FILE=$(osascript -e 'tell application "iTerm2" to activate' -e 'tell application "iTerm2" to set thefile to choose folder with prompt "Choose a folder to place received files in"' -e "do shell script (\"echo \"&(quoted form of POSIX path of thefile as Unicode text)&\"\")")
fi

if [[ $FILE = "" ]]; then
	echo Cancelled.
	# Send ZModem cancel
	echo -e \\x18\\x18\\x18\\x18\\x18
	sleep 1
	echo
	echo \# Cancelled transfer
else
	cd "$FILE"
	/usr/local/bin/rz -E -e -b --bufsize 4096
	sleep 1
	echo
	echo
	echo \# Sent \-\> $FILE
fi

```

#### 配置iTerm2客户端

> 打开路径：`iTerm2` -> `Preferences` -> `Profiles` -> `Advanced` -> `Triggers` -> `Edit`

配置如下图:

![iterm2-triggers](https://s2.loli.net/2022/02/11/WgLphE6uaOdSQc8.png)

**Regular Expression:**

> rz waiting to receive.\*\*B0100
>
> \*\*B00000000000000

### 3. 使用命令进行上传下载

> 注意：命令一定要在**容器内**操作，不要在本机操作。

#### sz命令发送文件到本地

执行完sz命令，本地iterm2会弹框选择下载到哪个目录

```shell
sz filename1            # 下载一个文件到本地
sz filename1 filename2  # 同时下载多个文件到本地
```

#### rz命令本地上传文件到容器

执行完rz命令，本地iterm2会弹窗让你选择需要上传的文件

```shell
rz    # 弹窗后选择需要上传的文件
rz -y # 如果上传的文件已存在不会上传，使用-y参数上传并覆盖原有文件
```


