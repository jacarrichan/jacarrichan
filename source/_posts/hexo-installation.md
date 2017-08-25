title: HEXO安装札记
date: 2014-08-24 17:59:57
category: hexo
tags: hexo
---
第一篇文章还是写给hexo，觉得是个坑。

<!--more-->

	   
	   
# 环境
Centos 6.5(Final) -64bit

## 安装nodejs 
### 编译工具
```
yum -y install gcc make gcc-c++ openssl-devel wget
```
### 下载源码及解压
```
wget http://nodejs.org/dist/v0.10.26/node-v0.10.26.tar.gz
tar -zvxf node-v0.10.26.tar.gz
```
###编译及安装
```
make && make install
```
### 验证是否安装配置成功
```
node -v   
```
### 配置nodejs的NPM镜像
除非有VPN，还是把这个配置上吧~~~   

```
npm config set registry https://registry.npm.taobao.org 
```

## 安装git
```
yum install  git
```

## 安装hexo

```
npm install -g hexo
```

-------------------



### 创建工程
```
hexo init
npm install 
npm install hexo-renderer-ejs --save
npm install hexo-renderer-stylus --save
npm install hexo-renderer-marked --save
```
最后三个是用来生成静态文件的插件，得安装。          



因系统安装在Vmware中，为了从宿主机访问页面便于测试，将hexo的监听改为0.0.0.0，即在[_config.yml]文件中配置端口号的附近增加[server_ip: 0.0.0.0]，注意有空格。   

### 生成静态文件并运行
```
hexo generate
hexo server
```


### 部署
编辑[_config.yml],将deploy部分的参数改成你自己的github page的地址   

```
deploy:
  type: github
  repository: git@github.com:jacarrichan/jacarrichan.github.io.git
```

执行如下命令部署到github      

```
hexo  d
```

### 备份 
1. 使用hexo-backup

1. 在_config.yml里面添加backup配置,repo的尾巴coding 表示分支


```
backup:
    type: git
    repository:
       origin: git@github.com:jacarrichan/jacarrichan.github.io.git,coding
```

1. 命令：

```
hexo backup --init #初始化备份 
hexo backup #备份操作 
```

### hexo 组件版本

```
C:\Users\Jacarri\blog>hexo version
hexo: 2.8.3
os: Windows_NT 6.3.9600 win32 ia32
http_parser: 2.3
node: 0.12.2
v8: 3.28.73
uv: 1.4.2-node1
zlib: 1.2.8
modules: 14
openssl: 1.0.1m
```
