# docker compose 部署 mysql 主从备份
## 使用方式
### 1、创建容器
``` shell
docker-compose up -d
```
### 2、mysql master docker 配置
1、进入 docker 容器
``` shell
docker-compose exec mysql-master bash
```
2、连接数据库
``` shell
   mysql -u root -p123456
```
3、创建 slave 用户，配置对应权限
在 mysql shell 里执行下列命令
``` shell
GRANT replication slave, replication client on *.* to 'slave'@'%' IDENTIFIED by "123456";
flush privileges; # 刷新权限
```
这个命令是为 MySQL 主从备份设置权限的命令。详解如下：
1. grant：这个关键字用于授权权限给指定的用户。
2. replication slave：这是一个 MySQL 权限，允许用户作为从服务器（slave）连接到主服务器（master）进行复制操作。
3. replication client：这也是一个 MySQL 权限，允许用户作为客户端连接到主服务器，以便查询复制状态和事件。
4. on *.*：表示授权的数据库和表的范围。在这种情况下，*.* 表示所有数据库和表。
5. to 'slave'@'%'：表示将权限授予用户名为 'slave'，在任何主机上都可以连接（由 '%' 表示）。
6. identified by "123456"：指定连接该从服务器（slave）时使用的密码。

可以在 mysql.user 表里查看刚刚创建的 slave 用户权限。

### 3、mysql slave docker 配置
1、进入 docker 容器
``` shell
docker-compose exec mysql-slave bash
```
2、连接数据库
``` shell
mysql -u root -p123456
```
3、配置备份服务
``` shell
change master to master_host='mysql-master',master_user='slave',master_password='123456',master_port=3306,master_log_file='mysql-bin.000003', master_log_pos=154,master_connect_retry=30;
```
这个命令用来配置连接 master 服务器的对应设置，详解如下：
1. change master to：这是一个 MySQL 命令，用于更改主从复制的配置。
2. master_host='mysql-master'：指定主服务器的主机名或 IP 地址。在这里，主服务器的主机名被设置为 'mysql-master'。
3. master_user='slave'：指定连接主服务器所使用的用户名。在这里，用户名设置为 'slave'。
4. master_password='123456'：指定连接主服务器所使用的密码。在这里，密码设置为 '123456'。请注意，为了安全起见，应该使用更强大的密码。
5. master_port=3306：指定主服务器监听连接的端口号。在这里，端口号设置为 3306。如果主服务器使用的是默认端口，则可以省略这个参数。
6. master_log_file='mysql-bin.000004'：指定主服务器二进制日志文件的文件名。这是用于在从服务器上开始复制的位置。在这里，文件名设置为 'mysql-bin.000004'。请注意，实际的二进制日志文件名可能不同，需要根据实际情况进行设置。
7. master_log_pos=154：指定从服务器开始复制的二进制日志文件位置。在这里，位置设置为 154。请注意，实际的日志位置可能不同，需要根据实际情况进行设置。
8. master_connect_retry=30：指定从服务器在连接到主服务器失败后重新尝试连接的时间间隔（以秒为单位）。在这里，重新尝试间隔设置为 30 秒。
开启备份服务
``` shell
start slave;
```
注意， master_log_file 与 master_log_pos 这两个参数可能不同，可以通过 show master status; 进行查看主服务的对应配置，然后进行修改。
检查 slave 状态
``` shell
show slave status;
```
如果看到 Slave_IO_Running: Yes，Slave_SQL_Running: Yes 表示已经成功开启主从。