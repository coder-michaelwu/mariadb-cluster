# mariadb-cluster
> 这个镜像是参考[LookBack博客文章](https://www.dwhd.org/20151202_220310.html)对MariaDB官方镜像的制作过程做简单修改和扩展，保留了[官方镜像](https://hub.docker.com/_/mariadb/)原有的使用方法和启动参数，仅仅是针对利用Docker镜像创建MariaDB主从复制集群的场景做了些调整。除过官方镜像的常规玩法外，该镜像还可以像下面这样玩：

基于本仓库制作镜像下载：
```
docker pull michaelwu2018/mariadb-cluster:10.1.37
```
官方玩法链接：https://hub.docker.com/_/mariadb/
## 1. 和官方镜像一样正常启动
```
docker run -d \
--name some-mariadb \
-e TIMEZONE=Asia/Shanghai \
-e MYSQL_ROOT_PASSWORD=123456 \
-p 3306:3306 \
michaelwu2018/mariadb-cluster:10.1.37
```
## 2. 作为Master节点启动
首先创建一个docker network
```
docker network create mariadb-cluster
```
然后再执行：
```
docker run -d \
--name mariadb-master \
--network mariadb-cluster \
-e TIMEZONE=Asia/Shanghai \
-v /data/mariadb-master:/data/mariadb \
-e MYSQL_ROOT_PASSWORD=123456 \
-e MASTER=1 \
-e MYSQL_REPLICATION_USER=repl_user \
-e MYSQL_REPLICATION_PASSWORD=123456 \
-p 3306:3306 \
michaelwu2018/mariadb-cluster:10.1.37
```
## 3. 作为Slave节点启动
```
docker run -d \
--name mariadb-slave \
--network mariadb-cluster \
-e TIMEZONE=Asia/Shanghai \
-v /data/mariadb-master:/data/mariadb \
-e MYSQL_ROOT_PASSWORD=123456 \
-e SLAVE=1 \
-e MYSQL_REPLICATION_USER=repl_user \
-e MYSQL_REPLICATION_PASSWORD=123456 \
-e MYSQL_MASTER_HOST=mariadb-master \
-e MYSQL_MASTER_PORT=3306 \
-e MASTER_LOG_FILE=mysqld-bin.000005 \
-e MASTER_LOG_POS=328 \
michaelwu2018/mariadb-cluster:10.1.37
```
## 4. 利用docker-compose.yml创建主从复制集群
```
version: '3.1'
services:
  mysql-master:
    environment:
      TZ: "Asia/Shanghai"
      MASTER: "1"
      MYSQL_ROOT_PASSWORD: "123456"
      MYSQL_REPLICATION_USER: "repl_user"
      MYSQL_REPLICATION_PASSWORD: "123456"
      MYSQL_DATABASE: "mydb"
    image: michaelwu2018/mariadb-cluster:10.1.37
    ports:
      - "3306:3306"
    restart: unless-stopped
    hostname: mysql-master

  mysql-slave:
    environment:
      TZ: "Asia/Shanghai"
      SLAVE: "1"
      MYSQL_ROOT_PASSWORD: "123456"
      MYSQL_REPLICATION_USER: "repl_user"
      MYSQL_REPLICATION_PASSWORD: "123456"
      MYSQL_MASTER_HOST: "mysql-master"
      MYSQL_MASTER_PORT: "3306"
      MASTER_LOG_FILE: "mysqld-bin.000005"
      MASTER_LOG_POS: "328"
      MYSQL_DATABASE: "mydb"
    image: michaelwu2018/mariadb-cluster:10.1.37
    links:
      - mysql-master
    restart: unless-stopped
    hostname: mysql-slave
```
## 5. 新增参数解释
和官方镜像同名的参数意义不变，这里仅对新增参数做简单说明。
### MASTER
使用`MASTER=1`标识镜像以Master节点启动，需要配合`MYSQL_REPLICATION_USER`和`MYSQL_REPLICATION_PASSWORD`两个参数使用
### SLAVE
使用`SLAVE=1`标识镜像以Slave节点启动，需要配合`MYSQL_REPLICATION_USER`、`MYSQL_REPLICATION_PASSWORD`、`MYSQL_MASTER_HOST`、`MYSQL_MASTER_PORT`、`MASTER_LOG_FILE`和`MASTER_LOG_POS`六个参数使用
### MYSQL_REPLICATION_USER
主从复制用户的用户名
### MYSQL_REPLICATION_PASSWORD
主从复制用户的用户密码
### MYSQL_MASTER_HOST
Master节点IP或Host
### MYSQL_MASTER_PORT
Master节点端口
### MASTER_LOG_FILE
主从同步起始文件名称，默认可以使用`mysqld-bin.000005`作为其值，具体视情况而定。
### MASTER_LOG_POS
主从同步起始文件位置，默认可以使用`328`作为其值，具体视情况而定。

## 6. 其他说明
镜像启动时会向`/etc/mysql/conf.d/mariadb.cnf`文件配置随机的`server-id`，小的mariadb复制集群应该不用考虑其会重复的问题，也不用配置`server-id`。
