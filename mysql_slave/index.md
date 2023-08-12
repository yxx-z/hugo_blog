# mysql 开启主从

mysql 开启主从
<!--more-->
##### 主服务器
编辑主服务器的my.cnf
[mysqld]下面添加
```
# 开启日志
og-bin=mysql-bin
# 服务器id 主服务器id不可相同
server-id=1
# 日志模式行
binlog_format=row
# mysql8及以后版本可开启 并行复制 数值范围正常在2-4
slave_parallel_workers =2
```

进入主服务器
执行
```
show master status;
```

会得到
```
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000006 |      601 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```
记住`File`和`Position`对应的值，后面从服务器中要用到。

##### 从服务器
编辑mysql从服务器的my.cnf
[mysqld]下添加
```
log-bin=mysql-bin
server-id=2
```
***

进入mysql从服务器
执行
```
CHANGE MASTER TO
MASTER_HOST='主服务器ip',
MASTER_PORT=主服务器端口,
MASTER_USER='主服务器账号',
MASTER_PASSWORD='主服务器密码',
#主服务器中执行show master status后得到的file
MASTER_LOG_FILE='mysql-bin.000006',
MASTER_LOG_POS=主服务器Position;
```

然后执行
```
start slave;
```
最后检查是否运行成功
运行下面代码
```
show slave status\G;
```
查看 `Slave_IO_Running` 和 `Slave_SQL_Running`的值是否是yes
如果不是 查看报错日志 `Last_Error`

写到这，忘记一件事。前面主服务器开了并行复制。这里从服务器也要设置下
运行代码
```
#停止复制进程
stop slave; 
#设置并行复制类型
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK'; 
重新启动复制进程
START SLAVE; 
```

##### 补充
如果主服务器重启后，日志名(例如 mysql-bin.000006)会变，这时候需要到从服务器更改主服务器的日志名
运行代码
```
#停止复制进程
STOP SLAVE;  
#使用新的文件名和位置
CHANGE MASTER TO MASTER_LOG_FILE = 'new_log_file_name', MASTER_LOG_POS = new_log_pos;
#重新启动复制进程
START SLAVE; 
```
替换 'new_log_file_name' 和 new_log_pos 为从主服务器 SHOW MASTER STATUS; 中获得的值。

---

> 作者: map[avatar:<nil> email:<nil> link:<nil> name:<nil>]  
> URL: http://localhost:1313/mysql_slave/  

