# 瑞吉外卖部署优化(普通版)


瑞吉外卖存在一些优化上的问题，因此进行以下方式的优化.

- redis缓存，可以使用使用redis保存登陆信息,share-jdbc.

- mysql数据库主从模式，读写分离.

- 试图将前后端完全分离，但个人认为貌似并没有完全的分离.

## 前端部署

- 启使用docker运行nginx启动.

## 后端部署

- 使用docker运行mysqlq.

## 部署细节(极为重要)

- 启动mysql+redis

```bash
docker-compose up -d
```

- 配置redis序列化方式，不使用object默认的序列化方式.

```bash
RedisConfig.java
```

- 主从模式&读写分离

```bash
# 1. 主数据库操作,配置登陆到该数据库的用户名root_slave,密码slave123456.
mysql -h 127.0.0.1 -u root -P 3307 -pabc123456
source db_reggie.sql

# 2.1 似乎MySQL的版本是5.7的，在8.0版本上无法运行.
# grant replication slave on *.* to 'rootslave'@'%' identified by 'slave123456';

# 2.2 尝试其他方式：创建用户root_slave，密码slave123456，'%'代表可以远程访问,'localhost'代表只能本地访问,'192.168.1.%'局域网访问，注意网段.
create user 'root_slave'@'%' identified by 'slave123456';

# 2.3 给新增用户赋权.
GRANT REPLICATION SLAVE ON *.* TO 'root_slave'@'%';

flush privileges;

show master status;

# 从数据库操作.
# 1. 登陆从数据库
mysql -h 127.0.0.1 -u root -P 3308 -pabc123456

# 2. 作为主数据库的从数据库,127.0.0.1报错2003.
change master to
master_host='91.199.209.120',
master_user='root_slave',
master_port=3307,
master_password='slave123456',
master_log_file='mysql-bin.000003',
master_log_pos=68986;

# 开启从数据库.
flush privileges;
start slave;

# 参看状态.
show master status;

# 没有分号.
show slave status\G
```

mysql -h 127.0.0.1 -u root_slave -P 3307 -pslave123456

```xml
<!-- 读写分离框架,写操作master,查询操作slave. -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
    <version>4.0.0-RC1</version>
</dependency>
```

## 登陆

- 前端路径

```bash
http://localhost:8080/front/index.html
```

- 后端路径

```bash
http://localhost:8080/backend/index.html
```

- 后端路径

```bash
http://localhost:8090
```

- 前端+后端

从使用上前后端并没有严格意义的分离。严格意义的分离应该是后端不该有任何的前端页面，只提供展示数据...

根据视频最后一章节是191，需要启动前端，启动后端。实际上由于已经嵌套了部分点餐的后端到nginx,所以启动的089B7DEE.pngApache实际上只是用户的点餐使用页面。