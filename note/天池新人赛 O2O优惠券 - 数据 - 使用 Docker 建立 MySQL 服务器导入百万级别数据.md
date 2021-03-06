# 天池新人赛 O2O优惠券 - 数据 - 使用 Docker 建立 MySQL 服务器导入百万级别数据

```
下载数据文件后，为了方便统计、分析以及特征处理，
在技术选型上考虑使用 RMDB 来进行数据储存，
并考虑结合使用 Docker 的 MySQL 镜像。
但请注意， 这并不是一篇关于 Docker 的使用教程。
```

`如无说明，以下命令均是在命令行下执行。`

`命令的详细用法可使用 docker COMMAND --help 查看`

0. [安装 Docker 以及下载 MySQL 镜像](#p0)
1. [启动容器，进入容器，连接数据库](#p1)
2. [使用第三方客户端连接 MySQL](#p2)
3. [创建数据库，创建数据表，导入数据](#p3)
4. [常见问题](#p4)


<hidden id="p0"/>

## 0. 安装 Docker 以及下载 MySQL 镜像

### 安装 Docker

我使用的开发环境是 MacOS , 所以直接在官网直接下载 dmg 文件直接安装，其他环境的安装请查看官方文档或其他相关教程，不再赘述。

查询命令 `docker -v`，可查看当前使用的 Docker 版本
```
Docker version 17.12.0-ce, build c97c6d6
```

### 下载 MySQL 镜像

打开命令行终端，输入以下命令

`docker pull mysql`

命令说明
```
docker pull
    - 启动 Docker 程序 下载镜像
mysql
    - MySQL 镜像，此处并不指定版本号，因此使用最新版本(即 latest)
```
下载完后，可以输入命令查看
`docker images`
```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mysql               latest              a8a59477268d        4 weeks ago         445MB
```

<hidden id="p1"/>

## 1. 启动容器，进入容器，连接数据库

### 思路整理

在启动 MySQL 容器之前，先想想我们想要达到的目的

1. 启动容器，并可以登录进容器，进行命令行的访问和操作
2. 可以使用第三方客户端（例如 Workbench）远程连接
3. 可以运行命令批量导入大量数据（百万级别）
4. 希望数据持久化，关闭容器后，数据并不会丢失

OK， 差不多想要的事情就是这样子，让我们一步一步来实现。

### 启动容器

首先，启动容器的命令如下

`docker run -d -p 127.0.0.1:3306:3306 --name mysqldb -v /development/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root mysql:latest`

命令说明
```
docker run
    - 运行镜像来创建并启动 MySQL 容器
-d
    - 使用后台模式运行容器（即 container）
-p 127.0.0.1:3306:3306
    - 发布 容器的 3306 端口（MySQL 端口）到宿主机 127.0.0.1（即运行容器的系统，也就是我的这台 Mac）的 3306 端口，
    也就是说使用宿主机作为跳板/桥接，客户端->(请求访问)宿主机3306端口->(实际访问)容器的3306端口
--name mysqldb
    - 容器的名字为 mysqldb
-v /development/mysql/data:/var/lib/mysql
    - 为了让数据持久化，把容器内的文件夹/var/lib/mysql挂载到宿主机的/development/mysql文件夹
-e MYSQL_ROOT_PASSWORD=root
    - 设置 MySQL 的环境变量， 这里即 用户 root 密码为 root
mysql:latest
    - 使用的 MySQL 镜像，若需要是使用指定版本，把 lasest 改为对应的版本号（前提是已把对应镜像下载）
```

使用命令 `docker container ls -a` 可查看所有容器,
或使用 `docker ps`查看在运行的容器，获取容器 ID
```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                      NAMES
768379a7f043        mysql:latest        "docker-entrypoint.s…"   3 days ago          Up 4 seconds        127.0.0.1:3306->3306/tcp   mysql
```
其他相关命令
> 启动容器 `docker start CONTAINER_ID`
>
> 重启容器 `docker restart CONTAINER_ID`
>
> 关闭容器 `docker stop CONTAINER_ID`
>
> 移除容器 `docker rm CONTAINER_ID`

### 进入容器

使用以下命令进入容器

`docker exec -i -t 768379a7f043 bash`

命令说明

```
docker exec
    - 运行容器中的命令
-i
    - 保持 STDIN（标准输入）打开
-t
    - 分配一个伪TTY链接，因为容器的本质就是一个根据所需功能最小化定制的 unix 系统
768379a7f043
    - 容器的 ID，
bash
    - 即 bash 命令， 可以使用 /bin/bash 代替
```

运行命令后即可以 root 身份进入容器系统
```
root@768379a7f043:/#
```

### 进入并连接 MySQL

进入容器后，我们需要启动 MySQL 终端来进入 MySQL

输入命令 `mysql -u root -p` 并输入密码 `root`

（有使用 unix 系统输入密码经验的同学应该知道这里任何输入都是不可见的）

```
root@768379a7f043:/#  mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.11 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
看来已经连接成功了。

<hidden id="p2"/>

## 2. 使用第三方客户端连接 MySQL

我们尝试使用第三方客户端来连接我们创建的 MySQL 数据库， 例如 MySQLWorkbench。

在 MySQLWorkbench 创建新连接并配置好后，点击 Test Connection , 居然弹出了错误信息
```
Failed to Connect to MySQL at 127.0.0.1:3306 with user root

Authentication plugin 'caching_sha2_password' cannot be loaded:
lopen(/usr/local/mysql/lib/plugin/caching_sha2_password.so, 2):
image not found
```

有问题自然找谷歌，看来是 MySQL 默认使用的加密方式是`caching_sha2_password`。



为了方便开发，我们需要把加密方式改为 `mysql_native_password`。

切换回 MySQL 终端，执行修改密码加密方式命令

`ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root';`

再查看 user 表, 发现已修改成功
```
mysql> select user, host,plugin , authentication_string from mysql.user;
+------------------+-----------+-----------------------+------------------------------------------------------------------------+
| user             | host      | plugin                | authentication_string                                                  |
+------------------+-----------+-----------------------+------------------------------------------------------------------------+
| root             | %         | mysql_native_password | *81F5E21E35407D884A6CD4A731AEBFB6AF209E1B                              |
| mysql.infoschema | localhost | mysql_native_password | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE                              |
| mysql.session    | localhost | mysql_native_password | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE                              |
| mysql.sys        | localhost | mysql_native_password | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE                              |
,)n7'sMO}/`Qx85'z1ul3za624iuvEAbIKX2B/MbmX74K7dkBpIHnZzso6. |05$m
+------------------+-----------+-----------------------+------------------------------------------------------------------------+
5 rows in set (0.00 sec)
```

再尝试点击 MySQLWorkbench 的 Test Connection, 这次弹出了成功链接信息
```
Successfully made the MySQL connection

Information related to this connection:

Host: 127.0.0.1
Port: 3306
User: root
SSL: enabled with DHE-RSA-AES256-SHA

A successful MySQL connection was made with
the parameters defined for this connection.
```
<hidden id="p3"/>

## 3. 创建数据库，创建数据表，导入数据

### 创建数据库， 数据表

创建数据库 coupon
``` sql
create database coupon;
```

根据数据文件，创建数据表为 ccf_offline_stage1_train ，为方便导入数据，所有字段的类型均为 varchar
``` sql
CREATE TABLE `ccf_offline_stage1_train` (
  `user_id` varchar(50) ,
  `merchant_id` varchar(50) ,
  `coupon_id` varchar(50) ,
  `discount_rate` varchar(50) ,
  `distance` varchar(50) ,
  `date_received` varchar(50) ,
  `date_consumption` varchar(50)
) ;
```

### 导入数据

#### 放弃使用 MySQLWorkbench 导入

在开始的时候， 我尝试使用 MySQLWorkbench 的 Import Data 功能。

但发现效率实在太低，速度为80条/秒, 导入了3小时多才不到1/4。如果数据量是在百万级别，则需3~4小时才能导入100万条数据。

#### LOAD DATA INFILE 登场

果然不少人有百万甚至千万级别的数据量一次性导入的需求。

转换策略，考虑改使用 MySQL 自身的 `LOAD DATA INFILE` 命令。

几经摸索之下，我需要做这么几件事
1. 复制数据文件到容器指定路径
2. 修改容器中 MySQL 配置的安全策略secure-file-priv，
3. 导入数据并验证

具体步骤

> 复制文件到容器
>>因为容器本身并无ssh 等功能，但使用 Docker 的复制命令可以轻松实现
>>
>> 登录容器
>> `docker exec -i -t 768379a7f043 bash`
>>
>> 创建好目标文件夹
>> `root@768379a7f043:/# mkdir mysql-files`
>>

> 修改 MySQL 配置文件
>>
>> 因为容器并未安装 vim ，因此需更新apt软件源并安装 vim
>>
>> 更新 apt, 命令： `apt-get update`
>>
>> 安装 VIM, 命令： `apt-get install vim`
>>
>> 修改配置, 命令： `vi /etc/mysql/my.cnf` 键盘按键 I 进入修改，
>>
>> 修改/添加 `secure-file-priv= /mysql-files/`
>>
>> 按 键盘按键 ESC 退出，并按“:” 输入 “wq” 保存修改
>>
>> 使用`exit`命令退出容器

> 重启MySQL容器
>> 命令 `docker restart 768379a7f043`

> 登入容器并进入 MySQL 终端
>> `docker exec -it 768379a7f043`
>>
>> `mysql -u root -p`

> 使用 LOAD DATA INFILE 命令加载数据
``` sql
-- 清空旧数据
TRUNCATE TABLE coupon.ccf_offline_stage1_train;
-- 导入数据到表 coupon.ccf_offline_stage1_train
LOAD DATA INFILE '/mysql-files/ccf_offline_stage1_train.csv'
INTO TABLE coupon.ccf_offline_stage1_train
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

结果使用了46.37秒便导入了175万多条数据。
```
Query OK, 1754884 rows affected (46.37 sec)
Records: 1754884  Deleted: 0  Skipped: 0  Warnings: 0
```
<hidden id="p4"/>

## 4. 常见问题

### 1）The MySQL server is running with the --secure-file-priv option

使用MySQL 命令 查看secure_file_priv 的路径是否为数据文件所在目录

`SHOW VARIABLES LIKE "secure_file_priv";`

如果为 NULL 即没有被允许的目录，参考文中《修改 MySQL 配置文件》部分进行配置

### 2）ERROR 1148 (42000): The used command is not allowed with this MySQL version

当使用命令 `LOAD DATA LOCAL INFILE`时发生，MySQL 基于安全机制，默认不允许使用 LOCAL，防止被恶意访问本地文件。

如需使用，对于搭建 MySQL服务器， 需使用源文件编译时添加 --enable-local-infile。

对于 Docker 的 MYSQL 镜像，我没有找到解决办法， 

即使在配置文件加入 `local-infile=1` 或登录时使用 `mysql --local-infile -u root -p` 也不生效。
如果你有解决方案，欢迎分享。

本文完。

