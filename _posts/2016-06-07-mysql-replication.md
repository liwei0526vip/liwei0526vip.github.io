---
layout: post
title:  "MySQL主从复制原理与实践"
categories: 数据库
tags: MySQL
author: 肖邦
---

* content
{:toc}




> MySQL 主从(MySQL Replication)，主要用于 MySQL 的实时备份、高可用HA、读写分离。在配置主从复制之前需要先准备 2 台 MySQL 服务器。

## 一、MySQL主从原理

* 1. 每个从仅可以设置一个主。
* 2. 主在执行 SQL 之后，记录二进制 LOG 文件(bin-log)。
* 3. 从连接主，并从主获取 binlog，存于本地 relay-log，并从上次记住的位置起执行 SQL，一旦遇到错误则停止同步。

## 二、Replication原理推论

* 1. 主从间的数据库不是实时同步，就算网络连接正常，也存在瞬间，主从数据不一致。
* 2. 如果主从的网络断开，从会在网络正常后，批量同步。
* 3. 如果对从进行修改数据，很可能从在执行主的bin-log出错而停止同步，一般不会修改从的数据。
* 4. 一个衍生的配置是双主，互为主从配置，只要双方的修改不冲突，可以工作良好。
* 5. 如果需要多主的话，可以用环形配置，这样任意一个节点的修改都可以同步到所有节点。
* 6. 可以应用在读写分离的场景中，用以降低单台 MySQL 服务器的 I/O。
* 7. 可以实现 MySQL 服务的 HA 集群。
* 8. 可以是一主多从，也可以是相互主从(主主)。

## 三、实验环境

* 操作系统：`CentOS 6.8_x64`
* MySQL 版本：`5.1.73`(主从版本要一致)
* MySQL 安装：`yum` 安装的方式
* 主 IP 地址：`192.168.0.8(master)`
* 从 IP 地址：`192.168.0.18(slave)`

## 四、主从的基本配置

* 1、对master的设置

修改 master 数据库的配置文件，vim /etc/my.cnf
```
[mysqld]
... ... ... ...
log-bin=mysql-bin    # 二进制日志名称，开启bin-log
server-id=8          # 为服务器设置一个独一无二的id，这里用IP的最后一位。
```
重启 master 数据库服务：
```
service mysqld restart
```

* 2、对slave的设置

对于 slave 的设置，不需要开启二进制日志，仅需要设置以下 server-id 即可。
```
server-id=18
```
重启从服务区器。

## 五、创建主从复制账号

为了让 slave 能够通过 master 来获取二进制日志，需要专门给 slave 创建一个用户 repl，在主上操作。

```sql
mysql> grant replication slave on *.* to 'repl'@'192.168.0.18' identified by '123456';
Query OK, 0 rows affected (0.00 sec)
```

## 六、查看主服务器BIN日志的信息

执行完之后记录下这两值，然后在配置完从服务器之前不要对主服务器进行任何操作，因为每次操作数据库时这两值会发生改变。

```sql
mysql> show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      259 |              |                  |
+------------------+----------+--------------+------------------+
```

## 七、设置从服务器并启用slave

从上执行如下代码：
```sql
mysql> change master to
    -> master_host="192.168.0.8",
    -> master_user="repl",
    -> master_password="123456",
    -> master_log_file="mysql-bin.000001",
    -> master_log_pos=248;
```
在从服务器配置完成，启动从服务器：
```sql
mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
```
查看主从设置是否成功：
```sql
mysql> show slave status\G;
... ... ... ...
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```
上面两项均为 yes，说明配置成功。

## 八、测试主从

在主节点上创建一个数据库 test 或一张表，然后在从节点上查看是否有 test 数据库或 table 表的创建。

 