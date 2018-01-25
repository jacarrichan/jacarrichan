---
title: 构建pinpoint的docker镜像
date: 2018-01-25 15:16:47
tags: [pinpoint,docker]
---
使用docker  build 命令构建pinpoint镜像

  <!-- more -->
## clone 代码
将代码(https://github.com/jacarrichan/docker-pinpoint)clone到本地。
##  构建镜像

在根目录分别执行如下4个命令，

```
[root@localhost docker-pinpoint]# 
docker build -t pinpoint-mysql:1.6.2 pinpoint-mysql
docker build -t pinpoint-hbase:1.6.2 pinpoint-hbase
docker build -t pinpoint-collector:1.6.2 pinpoint-collector
docker build -t pinpoint-web:1.6.2 pinpoint-web
docker build -t pinpoint-agent:1.6.2 pinpoint-agent

```
* -t表示打tag

pinpoint发行包是被github托管在aws的云存储上，所以在执行上述4个命令时，请祈祷网络OK。如果网络下载失败，那就重试相应的命令吧。

当你祈祷完毕后，我们会得带4个镜像，使用docker images命令查看，结果如下：

```
[root@localhost docker-pinpoint]# docker images
REPOSITORY                                    TAG                 IMAGE ID            CREATED             SIZE
pinpoint-web                                  1.6.2               3af29b3b3bee        15 seconds ago      662.8 MB
pinpoint-agent                                1.6.2               a9f48a57a1f4        2 hours ago         17.03 MB
pinpoint-collector                            1.6.2               bf7b224e3ac1        4 hours ago         607.9 MB
pinpoint-hbase                                1.6.2               dfafa9ce6392        4 hours ago         990.4 MB
pinpoint-mysql                                1.6.2               cc4b584f7798        6 hours ago         408.6 MB
```
## 放弃构建，拉取仓库镜像

如果你觉得你电脑网络实在是坑爹，那你可以拉取别人已经push到仓库的镜像


拉取中央仓库的命令如下
````
docker pull  wahyd4/pinpoint-mysql
docker pull  wahyd4/pinpoint-hbase
docker pull  wahyd4/pinpoint-collector
docker pull  wahyd4/pinpoint-web
docker pull  wahyd4/pinpoint-agent
````
