在平时的工作中，我们经常会遇到因为数据库压力比较大，造成整个系统的响应慢甚至直接不可用的情况。

造成数据库压力大的情况一般都是来源于```读操作```，针对这种问题，我们一般可以使用```读写分离```来让```写操作```和```读操作```分离开来，还可以根据压力情况，部署多个节点来降低读操作压力，提升系统性能。

要实现读写分离，就要先配置```主从同步```，让主库的数据同步到从库，下面让我们用 ```MySQL 5.7 ``` 来实验一下。


我们需要实现一个整库同步节点和一个单库同步节点，所以我们要准备一个 ```master``` 节点，两个 ```slave``` 节点。

#### 修改数据库配置文件
首先我们先修改 ```master``` 库的配置文件，开启 ```binlog```，设置 ```server-id```，配置如下：

```shell
[mysqld]
log-bin=mysql-bin
server-id=1
```

然后修改从库1 ```slave1``` 的配置，如下：
```shell
server-id=2
```

然后再修改从库2 ```slave2``` 的配置，因为从库2只同步一个库，所以我们指定需要同步的库，配置如下：
```shell
server-id=3
replicate-do-db=test1
```

这个 ``` server-id ``` 一定要是网络内唯一，不可重复。

配置完成后需要重启服务让配置生效。

#### 创建复制账户
然后我们创建一个用来同步复制数据的用户。

```shell
mysql > grant replication slave on *.* to 'reptest'@'%' identified by '123456';
```

其中 ``` replication ``` 代表这个用户只能用来进行主从复制。

#### 查看binlog信息
我们通过 ```show binary logs ``` 来查，然后记住 ``` Log_name ``` 和 ``` File_size ``` 两个值，或者使用 ``` show master status ```进行查看。

```shell
show binary logs;

+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |      1791 |
+------------------+-----------+
1 row in set (0.00 sec)
```


* 注意：**如果是生产数据库或者有数据产生，请在获取 binlog 信息之前加上锁，防止对数据库的任何写操作，然后在下一步之后释放锁.**


#### 导出SQL
使用 ```mysqldump```导出数据库，如果主库没有数据，可以忽略这一步。

```shell
root@xxxx:/# mysqldump -uroot -p --all-databases > /var/sql/mysql_bak.sql
```

这个时候，如果之前对数据库加了锁，现在就可以释放了。

然后将导出的 SQL 导入到 ```slave1``` 和 ```slave2```。

```shell
root@ssss:/# mysql -uroot -p < /var/sql/mysql_bak.sql
```

#### 在从库里面设置同步
```shell
mysql> CHANGE MASTER TO
    -> MASTER_HOST='172.25.0.2',
    -> MASTER_USER='reptest',
    -> MASTER_PASSWORD='123456',
    -> MASTER_LOG_FILE='mysql-bin.000001',
    -> MASTER_LOG_POS=1791;
```

* **MASTER_HOST** 是 主库IP
* **MASTER_USER** 是主库账号
* **MASTER_PASSWORD** 是主库账号的密码
* **MASTER_LOG_FILE** 是之前记录的主库 binlog 的 Log_name
* **MASTER_LOG_POS** 是之前记录的主库 binlog 的 File_size

在所有从服务器都配置好之后，把同步功能打开，使用 ```start slave```命令。

我们可以通过 ```show slave status \G;``` 查看，如果 ```Slave_IO_Running``` 和 ```Slave_SQL_Running``` 都是 ```Yes```，则说明配置成功了。

#### 同步测试
在主库创建 ```test1``` 和 ```test2``` 数据库，然后在 ```test1```数据库创建 ```t1``` 表，在 ``` test2 ``` 库创建 ```t2``` 表，对 ```t1``` 和 ```t2``` 各插入一条数据，然后观察 ```slave1``` 和 ```slave2``` 的变化。

先看```master```数据：
```
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test1              |
| test2              |
+--------------------+
6 rows in set (0.00 sec)
```

```
mysql> USE test1;
mysql> select * from t1;
+----+--------+
| id | text   |
+----+--------+
|  1 | 111111 |
+----+--------+
1 row in set (0.00 sec)
```

```
mysql> USE test2;
mysql> select * from t2;
+----+--------+
| id | text   |
+----+--------+
|  1 | t2-111 |
+----+--------+
1 row in set (0.00 sec)
```

可以看到主库里面```test1``` 和 ```test2``` 均有数据，我们在这两个表分别再插入一条数据，再去观察从库的变化。

先观察通过所有库的从库1：
```
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test1              |
| test2              |
+--------------------+
6 rows in set (0.00 sec)
```
可以看到，```test1``` 和 ```test2``` 都进行了同步，我们再进去看看数据。

```
mysql> USE test1;
mysql> select * from t1;
+----+--------+
| id | text   |
+----+--------+
|  1 | 111111 |
|  2 | 222222 |
+----+--------+
2 rows in set (0.00 sec)
```

```
mysql> USE test2;
mysql> select * from t2;
+----+--------+
| id | text   |
+----+--------+
|  1 | t2-111 |
|  2 | t2-222 |
+----+--------+
2 rows in set (0.00 sec)
```

从结果可以看到，从库1 ```slave1```确实同步了所有的数据库，成功！

我们再去看看从库2的结果：
```
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test1              |
+--------------------+
5 rows in set (0.01 sec)
```

确实只同步了 ```test1``` 数据库，和配置一样，进去看看数据。
```
mysql> USE test1;
mysql> select * from t1;
+----+--------+
| id | text   |
+----+--------+
|  1 | 111111 |
|  2 | 222222 |
+----+--------+
2 rows in set (0.00 sec)
```

数据也同步了过来，成功！


#### 结束
这里只是简单的对主从同步进行了介绍，想要更详细的了解主从相关内容可以查看阅读 ```MySQL``` 官方技术文档。

得益于 ```Docker```，我们可以快速的搭建测试环境并进行测试，测试环境我打包成了 ```docker-compose``` 放到 GitHub，如有需要可查看。

* [5.7版本主从同步文档](https://dev.mysql.com/doc/refman/5.7/en/replication.html)
* [8.0版本主从同步文档](https://dev.mysql.com/doc/refman/8.0/en/replication.html)