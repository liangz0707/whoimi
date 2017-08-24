# docker简介

docker可以构建Build、运输Ship、运行Run

docker是一个一次建立到处运行的平台

## Containers vs. VMs 

docker是容器  openstack是虚拟机

Docker组成 : Docker Client     / Docker Server

Docker组件：镜像Image（只读）  / 容器Container（对照JVM） / 仓库Repository（对照GitHub）

## 改变工程交付的方式

## Docker 的使用场景：

简化配置、提高开发效率、应用隔离、服务器整合、

Linux安装  Linux内置

```shell
sudo apt-get install docker.io
source  /etc/bash_completion.d/docker.io
```

启动容器

```shell
docker run -i(表示为容器始终打开标准输入) -t(为容器分布一个伪ttp终端)ubuntu /bin/bash
```

容器信息查看

```shell
docker ps [-a](所有容器) [-l](最新容器)  什么都不加表示正在运行中的容器
CONTAINER ID 容器的唯一ID             NAMES是docker守护进程分配的名字
docker inspect [容器的名字 或  ID]   显示容器详细的信息
```
定义名字

```shell
docker run --name=[容器名字] -i -t  ubuntu /bin/bash
```

重新启动已经停止的

```shell
docker start -i container01
```

删除已经停止的容器

```shell
docker rm  container01
```

始终运行的容器适用于在后台运行的服务  没有交互式回话

```shell
docker run -i -t ubuntu /bin/bash
```

使用 Ctrl-P  Ctrl-Q 的组合 退出交互式容器bash  这样容器会在后台运行

此时使用ps命令查看  就能看到  运行还在执行

回到正在执行的容器

```shell
docker attach [name or ID]
```

在运行的容器中使用 exit  完全退出

使用run命令启动守护时容器，会返回一个ID

```shell
docker run --name [A name] -d (表示后台的方式启动)  ubuntu(镜像名字) /bin/sh -c "死循环"
```

借助 logs 命令查看容器日志

```shell
docker logs [-f](动态跟踪)  [-t](加上实际) [--tail] 返回后面的n条
```

top 可以查看运行中容器内进程的情况

```shell
docker  top [name   or  ID]
```

使用exec 在已经运行的docker 容器中启动新进程

```shell
docker  exec [-d] [-i] [-t]  容器名 [COMMAND]  [ARG]
```

使用stop 或 kill 停止  运行中的容器

```shell
docker  stop [name or ID]
```
stop 是发送信号 等待容器的停止 ，停止后会返回容器的名字

kill 是直接杀死容器

```shell
docker  kill [name or ID]
```

使用  man 查看详细介绍

```shell
man  docker-run
```

[back](../../index.md)





