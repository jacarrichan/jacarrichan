---
title: 在CentOS7上安装Docker
date: 2018-01-26 12:02:51
tags: [CentOs,Docker]
---

在CentOS7 X86上安装新版Docker和Docker-compose ,并且处理权限问题。

  <!-- more -->
  
## 系统要求

为了安装docker，需要准备 64-bit的CentOS 7

删除非官方的Docker包

yum的仓库中有一个很旧的Docker包, 现在Docker官方已经将Docker更名为docker-engine. 如果你已经安装了这个版本的Docker需要使用下边的命令删除它

```
$ sudo yum -y remove docker docker-common container-selinux
```

/var/lib/docker 无需删除.

## 安装Docker

有两种方式对docker提供了安装。

### 使用yum方式

设置Docker仓库

使用下边的命令设置最新稳定版的docker仓库

```
$ sudo yum-config-manager \
    --add-repo \
    https://docs.docker.com/v1.13/engine/installation/linux/repo_files/centos/docker.repo
```
    

更新yum源

```
$ sudo yum makecache fast
```
安装最新版的docker

```
$ sudo yum -y install docker-engine
```

或者安装其他版本docker

```
$ yum list docker-engine.x86_64  --showduplicates |sort -r


docker-engine.x86_64  1.13.0-1.el7                               docker-main
docker-engine.x86_64  1.12.5-1.el7                               docker-main   
docker-engine.x86_64  1.12.4-1.el7                               docker-main   
docker-engine.x86_64  1.12.3-1.el7                               docker-main   
$ sudo yum -y install docker-engine-<VERSION_STRING>
```

启动docker

```
$ sudo systemctl start docker
$ sudo systemctl enable docker
```

为了确认docker安装运行正常安装一个demo镜像

```
$ sudo docker run hello-world
```

升级Docker

```
$ sudo yum makecache fast

$ yum list docker-engine.x86_64  --showduplicates |sort -r

docker-engine.x86_64  1.13.0-1.el7                               docker-main
docker-engine.x86_64  1.12.5-1.el7                               docker-main   
docker-engine.x86_64  1.12.4-1.el7                               docker-main   
docker-engine.x86_64  1.12.3-1.el7                               docker-main   
$ sudo yum -y install docker-engine-<VERSION_STRING>
```

###  rpm方式安装

访问https://yum.dockerproject.org/repo/main/centos/ 按照操作系统版本号选择对应的docker版本软件。

把path改成保存docker.rpm的目录

```
$ sudo yum -y install /path/to/package.rpm
```

启动docker

```
$ sudo systemctl start docker
$ sudo systemctl enable docker
```

为了确认docker安装运行正常安装一个demo镜像

```
$ sudo docker run hello-world
```

###  卸载docker

卸载docker软件

```
$ sudo yum -y remove docker-engine
```

镜像, 容器, volumes, 配置文件 都不会自动删除. 需要手动删除，如果确定不需要 可以执行以下命令:

```
$ sudo rm -rf /var/lib/docker
```


同时必须手动删除各种配置文件  

### 安装 Docker-compose

#### 使用pip
先：

```
yum install python-pip.noarch
```

对安装好的pip进行一次升级

```
sudo pip install --upgrade pip
pip install docker-compose
```

#### 直接下载安装

```
curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

```

* 如果不执行chmod,会导致出现[Permission denied]或者[sudo docker-compose: command not found]

最后docker-compose -version 查看


官方文档：https://docs.docker.com/engine/installation/linux/centos/



##  docker权限
启动docker,运行命令遇到问题

*** dial unix /var/run/docker.sock: permission denied.Are you trying to connect to a TLS-enabled ***

解决办法： 
把当前用户加入docker用户组。

```
$sudo gpasswd -a ${USER} docker
```

查看是否添加成功：

```
cat /etc/group | grep ^docker
```

重启docker

```
sudo serivce docker restart
```

到这步如果还不成功，logout当前用户，再login

