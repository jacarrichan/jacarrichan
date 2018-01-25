---
title: 运行pinpoint的docker镜像
date: 2018-01-25 15:16:47
tags: [pinpoint,docker]
---
使用docker-compose 和docker run命令运行pinpoint镜像

  <!-- more -->
## clone 代码
将代码(https://github.com/jacarrichan/docker-pinpoint)clone到本地。
如果在上文构建的镜像中clone，就不需要clone了。

##  运行镜像
首先
关于 docker compose配置地址：
https://github.com/jacarrichan/docker-pinpoint/blob/master/docker-compose.yml

1. 请确保docker-compose.yml里面image的配置类似[ image: pinpoint-*:1.6.2]; 
1. 如果你想使用wahyd4上传到中央仓库的镜像，那image的值应该类似[image: pinpoint-wahyd4:1.6.2]


说完了docker-compose.yml的配置，现在开始用这个文件把pinpoint-mysql、pinpoint-hbase、pinpoint-web、pinpoint-collector一起跑起来.

你需要在根目录执行[docker-compose up  -d]命令

```
[root@localhost docker-pinpoint]# docker-compose up  -d
Starting pinpoint-hbase ... 
Starting pinpoint-mysql ... done
Starting pinpoint-collector ... 
Starting pinpoint-web ... done
```

* -d 表示后台运行

### 查看后端启动日志

```
docker-compose logs
```

### 查看执行[docker-compose up  -d]命令创建的网络

```
[root@localhost docker-pinpoint]# docker  network ls
NETWORK ID          NAME                     DRIVER              SCOPE
c7e74958e34f        bridge                   bridge              local               
8ef31b5dd24a        demo_default             bridge              local               
fdb21735c9e2        dockerpinpoint_default   bridge              local               
7153abc7d7d9        host                     host                local               
26b87627e255        none                     null                local  

```

可以看到刚才创建的网络是***dockerpinpoint_default***

## 构建和运行demo应用的镜像

### 构建demo应用的镜像

在examples目录执行[docker build -t joinfaces-example-with-pinpoint-agent joinfaces-example-with-docker-image]

```
[root@localhost examples]# docker build -t joinfaces-example-with-pinpoint-agent joinfaces-example-with-docker-image/
Sending build context to Docker daemon 39.87 MB
Step 1 : FROM marcosamm/pinpointagent-oraclejre:latest
 ---> bc032b756885
Step 2 : COPY app/joinfaces-example/ /opt/joinfaces-example
 ---> a56158361f01
Removing intermediate container f4fa12c2e239
Step 3 : COPY app/pinpoint-agent/ /opt/pinpoint-agent
 ---> f9ddaacbc898
Removing intermediate container c90d39c35a96
Step 4 : EXPOSE 8080
 ---> Running in b6305ccc2fc3
 ---> 5c2c24fdb113
Removing intermediate container b6305ccc2fc3
Step 5 : ENTRYPOINT java -javaagent:/opt/pinpoint-agent/pinpoint-bootstrap.jar -Dpinpoint.agentId=dockerfile -Dpinpoint.applicationName=joinfaces-example -jar /opt/joinfaces-example/joinfaces-example-2.4.0-SNAPSHOT.jar
 ---> Running in d91d5dd84e9d
 ---> 85e064f7d7d1
Removing intermediate container d91d5dd84e9d
Successfully built 85e064f7d7d1
```
### 运行demo应用镜像

```
[root@localhost examples]# docker run -d  --expose=8080 -p 38080:8080 --link=pinpoint-collector:collector   --net dockerpinpoint_default joinfaces-example-with-pinpoint-agent
073340717e14f17b996b2af411550cfa52ccb65a1c573f479e8e18b56c914147
[root@localhost examples]# docker logs 073340717e14f17b996b2af411550cfa52ccb65a1c573f479e8e18b56c914147

```

1. [-p 38080:8080]表示把内部8080端口映射到宿主机的38080端口上去；
1. [--link=pinpoint-collector:collector]   表示把容器名为pinpoint-collector映射成hostname是collector上，便于pinpoint.config文件里面的[profiler.collector.ip]使用

###  查看应用监控

1. 访问[http://hostip:28080] 查看pinpoint-web的界面
1. 访问[http://hostip:38080] 查看demo应用的界面

##  其他命令

1. 查看 镜像列表：[docker images]
1. 查看 容器列表：[docker ps -a ]
1. 删除 镜像：[docker rmi  $image_id]
1. 删除 容器：[docker rmi  $container_id]
1. 进入 容器：[docker exec -it   $container_id /bin/bash]. 之后就可以进去容器里面的文件了

## 可能出现的问题

### 第二次重启容器，pinpoint-web上可以看到demo应用，但是看不到访问记录；

请查看hbase启动的日志，看有没有出现这样的异常[java.io.IOException: Directory /home/pinpoint/hbase/WALs/0afcbb063a9b,44027,1516902955987-splitting is not empty]。如果有的话，可以删除docker volumes目录对应的文件，命令为[rm -rf  /var/lib/docker/volumes/dockerpinpoint_hbase_data/_data/WALs/]，然后把容器容器[docker-compose down & docker-compose up -d]
