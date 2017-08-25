title: 在Centos上安装MySQL
date: 2014-08-24 18:42:19
category: database
tags: [mysql, centos]
---


Linux系统上安装MySQL 5.5prm

### 1.准备工作
``` download
从MySQL官网上分别下载mysql服务器端于客户端包:
http://cdn.mysql.com/Downloads/MySQL-5.1/MySQL-server-5.1.73-1.glibc23.x86_64.rpm
http://cdn.mysql.com/Downloads/MySQL-5.1/MySQL-client-5.1.73-1.glibc23.x86_64.rpm
```

### 2.检测系统是否安装MySQL
1)查找安装信息

- rpm -qa | grep -i mysql

若已安装过，会出现以下信息

``` 
MySQL-server-5.1.73-1.glibc23.x86_64
MySQL-client-5.1.73-1.glibc23.x86_64
```
2)那么输入以下命令删除它：

- rpm -ev MySQL-server-5.0.22-0.i386
- rpm -ev MySQL-client-5.0.22-0.i386


### 3.安装MySQL
第一步：安装mysql服务端,输入以下命令

- rpm -ivh MySQL-server-5.1.73-1.glibc23.x86_64.rpm

当出现如下信息表示安装成功：

``` 
Preparing...　　　　　　　########################################### [100%]
1:MySQL-server　　　　　########################################### [100%]
。。。。。。（省略显示）
 /usr/bin/mysqladmin -u root password 'new-password'
/usr/bin/mysqladmin -u root -h test1 password 'new-password'
。。。。。。（省略显示）
```
第二步：检测mysql 3306是否安打开，输入以下命令

- netstat -nat
当出现如下时，表示mysql 3306端口打开,MySQL服务已经启动

``` 
Active Internet connections (servers and established) 
Proto Recv-Q Send-Q Local Address Foreign Address State 
tcp00 0.0.0.0:3306 0.0.0.0:* LISTEN 
```
第三步：安装mysql客户端

- rpm -ivh MySQL-client-5.1.73-1.glibc23.x86_64.rpm
当出现如下：表示安装成功

``` 
warning: MySQL-client-5.1.7-0.i386.rpm: V3 DSA signature: NOKEY, key ID 5072e1f5 
Preparing...########################################### [100%] 
1:MySQL-client ########################################### [100%] 
```
### 4.配置MySQL
1)上面都是安装完成了，但都是默认的，还需要很多配置。
默认安装位置及作用

``` 
1.数据库目录 /var/lib/mysql/ 

2、配置文件 
/usr/share/mysql（mysql.server命令及配置文件） 

3、相关命令 
/usr/bin(mysqladmin mysqldump等命令) 

4、启动脚本 
/etc/rc.d/init.d/（启动脚本文件mysql的目录）
    如：/etc/rc.d/init.d/mysql start/restart/stop/status
```
2）由于MySQL数据库目录占用磁盘比较大，所以我在/根目录下建了个个目录data，命令如下：

- mkdir /data

3）把数据库移动到data目录中去。输入以下命令：

- mv  /var/lib/mysql  /data

最后，进入data目录就会看到有一个mysql文件夹。
 
拷贝完后还需修改/etc/rc.d/init.d/mysql的datadir目录值，修改结果如：

``` 
basedir=
datadir=/data/mysql
```
4)拷贝配置文件到/etc目录下，并命名为my.cnf(必须名为my.cnf)

- cp  /usr/share/mysql/my-medium.cnf  /etc/my.cnf

这儿要注意：/usr/share/mysql/下有好几个结尾为cnf的文件，它们的作用分别是：

``` 
1.my-small.cnf是为了小型数据库而设计的。不应该把这个模型用于含有一些常用项目的数据库。
2.·my-medium.cnf是为中等规模的数据库而设计的。如果你正在企业中使用RHEL,可能会比这个操作系统的最小RAM需求(256MB)明显多得多的物理内存。由此可见，如果有那么多RAM内存可以使用，自然可以在同一台机器上运行其它服务。
3·my-large.cnf是为专用于一个SQL数据库的计算机而设计的。由于它可以为该数据库使用多达512MB的内存，所以在这种类型的系统上将需要至少1GB的RAM,以便它能够同时处理操作系统与数据库应用程序。
4·my-huge.cnf是为企业中的数据库而设计的。这样的数据库要求专用服务器和1GB或1GB以上的RAM。
这些选择高度依赖于内存的数量、计算机的运算速度、数据库的细节大小、访问数据库的用户数量以及在数据库中装入并访问数据的用户数量。随着数据库和用户的不断增加，数据库的性能可能会发生变化。
```

5)最后配置/etc/my.cnf文件的datadir,和mysql.sock路径以及默认编码utf-8.

```
[client]
password        = 123456
port            = 3306
socket          = /data/mysql/mysql.sock
default-character-set=utf8
 #Here follows entries for some specific programs
 
 #The MySQL server
[mysqld]
port            = 3306
socket          = /data/mysql/mysql.sock
skip-external-locking
key_buffer_size = 16M
max_allowed_packet = 1M
table_open_cache = 64
sort_buffer_size = 512K
net_buffer_length = 8K
read_buffer_size = 256K
read_rnd_buffer_size = 512K
myisam_sort_buffer_size = 8M
character_set_server=utf8
collation-server=utf8_general_ci
lower_case_table_names=1
character_set_client=utf8
 #(注意linux下mysql安装完后是默认：区分表名的大小写，不区分列名的大小写；lower_case_table_names = 0    0：区分大小写，1：不区分大小写)
max_connections=1000 #(设置最大连接数，默认为 151，MySQL服务器允许的最大连接数16384； )
[mysql]
 
default-character-set = utf8
 
no-auto-rehash

```
6）进入/usr/bin目录下重启mysql服务

- mysql_install_db
- cd   /usr/bin/mysql restart

7)登录mysql,Enter password:(直接回车，因为第一次为空密码)

- cd /usr/bin/mysql -u root -p

8）登录成功后，修改密码
进入>mysql环境下，
输入：

``` command
mysql> show databases;

+--------------------+
| Database           |
+--------------------+
| rmation_schema |
| mysql              |
| performance_schema |
| test               |
+--------------------+
4 rows in set (0.00 sec)

> mysql> use mysql;

Database changed

mysql> show tables;

+---------------------------+
| Tables_in_mysql           |
+---------------------------+
| columns_priv              |
| db                        |
| event                     |
| func                      |
| general_log               |
| help_category             |
| help_keyword              |
| help_relation             |
| help_topic                |
| host                      |
| ndb_binlog_index          |
| plugin                    |
| proc                      |
| procs_priv                |
| proxies_priv              |
| servers                   |
| slow_log                  |
| tables_priv               |
| time_zone                 |
| time_zone_leap_second     |
| time_zone_name            |
| time_zone_transition      |
| time_zone_transition_type |
| user                      |
+---------------------------+
24 rows in set (0.00 sec)

mysql>update user set password=password('123456')where user='root';
```
修改root密码为123456,最后重启mysql，密码生效

- service mysql restart

9）MySQL安装成功
10) 更改linux中mysql 3306的访问权限

修改iptables文件：

- vim /etc/sysconfig/iptables

增加：

``` 
-A RH-Firewall-1-INPUT -m state –state NEW -m tcp -p tcp –dport 3306 -j ACCEPT
```



