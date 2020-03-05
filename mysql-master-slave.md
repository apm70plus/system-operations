## 配置mysql主从  

**配置主服务器**  
在主服务器上，您必须启用二进制日志记录并配置唯一的服务器ID。编辑主服务器的配置文件 /etc/my.cnf，添加如下内容  
```
[mysqld]
log-bin=/var/log/mysql/mysql-bin
server-id=1
```

创建日志目录并赋予权限  
```
mkdir /var/log/mysql
chown mysql:mysql /var/log/mysql
```

重启服务  
```
systemctl restart mysqld
```

**注意：**  
```
如果省略server-id（或将其显式设置为默认值0），则主服务器拒绝来自从服务器的任何连接。

为了在使用带事务的InnoDB进行复制设置时尽可能提高持久性和一致性，
您应该在master my.cnf文件中使用以下配置项：
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1

确保未在复制主服务器上启用skip-networking选项。
如果已禁用网络，则从站无法与主站通信，并且复制失败。
```

创建一个仅具有复制过程权限的单独帐户，以最大程度地降低对其他帐户的危害。master上执行下面的操作  
```
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '123';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
mysql>
```

**配置从服务器**  
编辑其配置文件 /etc/my.cnf 并添加如下内容：  
```
[mysqld]
server-id=2
```

重启服务  
```
systemctl restart mysqld
```

查看主服务器的二进制日志的名称  
```
mysql> FLUSH TABLES WITH READ LOCK;
mysql> show master status \G

****************** 1. row ****************
             File: mysql-bin.000001
         Position: 0
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
```

在从服务器上配置连接到主服务器的相关信息，MASTER_HOST，MASTER_LOG_FILE要根据实际情况修改  
```
mysql> CHANGE MASTER TO
MASTER_HOST='mysql-master1',
MASTER_USER='repl',
MASTER_PASSWORD='123',
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=0;

mysql> start slave;
```
