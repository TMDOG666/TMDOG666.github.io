## PostgreSQL 笔记：PostgreSQL 主从复制

在现代应用程序中，数据库的高可用性和扩展性是至关重要的。PostgreSQL 提供了主从复制功能，可以在多个数据库实例之间复制数据，以实现冗余和负载均衡。本文将介绍如何在 Docker 环境中构建PostgreSQL 主从复制环境。

### 1. 主从复制原理

PostgreSQL 的主从复制是通过将主服务器的 WAL（Write-Ahead Logging）日志复制到从服务器实现的。以下是主要原理：

- **预写式日志（WAL）**： PostgreSQL 使用预写式日志记录事务更改，确保在任何时刻数据的完整性。在执行写入操作之前，系统会将更改先写入 WAL，这样即使发生崩溃，也可以通过 WAL 恢复数据。

- **主服务器（Master）**: 负责处理所有写入操作。当数据被写入时，它首先记录到 WAL 中，然后再应用到数据文件。
- **从服务器（Slave）**: 被配置为从主服务器接收和应用 WAL 日志，以保持数据一致性。它可以处于热备份状态，随时接收主服务器的更新。
- **复制角色**: 从服务器需要一个具备复制权限的角色（如 `repl`），通过此角色进行身份验证和连接。
- **异步与同步复制**: PostgreSQL 支持异步和同步复制。异步复制可以提高性能，但主服务器不会等待从服务器确认接收到数据，而同步复制则确保数据在主服务器和从服务器之间一致性。

### 2. 创建网络环境

首先，我们需要为 PostgreSQL 实例创建一个 Docker 网络，以便它们可以相互通信。

```bash
docker network create pg-network
```

![image-20241102193301802](https://github.com/user-attachments/assets/feb8b1ee-a07b-422a-9809-90c15fc333dd)


### 3. 创建主服务器

接下来，启动一个 PostgreSQL 主服务器容器。我们将数据存储在宿主机上，以便在容器重启时数据不会丢失。

```bash
docker run --network=pg-network --name pgsmaster -p 5500:5432 -e POSTGRES_PASSWORD=123456 -v /var/lib/pgsmaster:/var/lib/postgresql/data -d postgres:16.4
```


![image-20241102193328093](https://github.com/user-attachments/assets/1f9efb03-7840-41c9-ae8d-429988c9b032)

### 4. 创建从属服务器

同样，我们创建一个 PostgreSQL 从属服务器容器。它将用于接收主服务器的数据复制。

```bash
docker run --network=pg-network --name pgsslave -p 5501:5432 -e POSTGRES_PASSWORD=123456 -d postgres:16.4
```

![image-20241102193347793](https://github.com/user-attachments/assets/e23f32ab-0330-4dfe-b137-e0b34f3fbfb2)

![image-20241102193423517](https://github.com/user-attachments/assets/13c0ce75-6a3d-47eb-a2a5-8fde67104c96)

### 5. 获取 IP 地址

为了配置主从复制，我们需要获取主从服务器的 IP 地址：

```bash
docker inspect pgsmaster | grep IPAddress
docker inspect pgsslave | grep IPAddress
```

![image-20241102193457601](https://github.com/user-attachments/assets/8cf4c7e7-01f1-43a6-bbca-7f482d3f747d)


### 6. 配置主服务器

编辑主服务器的 `postgresql.conf` 文件，添加从属连接信息。使用上一步获取的 IP 地址。

```bash
cat >> /var/lib/pgsmaster/postgresql.conf <<-'EOF'
primary_conninfo = 'host=<主服务器IP> port=5432 user=repl password=repl' 
EOF
```

![image-20241102193615359](https://github.com/user-attachments/assets/556c8d76-3156-442c-b692-0f1eb00c14c4)


### 7. 更新 pg_hba.conf

为了允许从属服务器连接到主服务器，我们需要更新 `pg_hba.conf` 文件：

```bash
cat >> /var/lib/pgsmaster/pg_hba.conf <<-'EOF'
host	replication	repl		<从属服务器IP>/32		md5
EOF
```

![image-20241102194049793](https://github.com/user-attachments/assets/6b2768a2-1239-4380-83ac-c3f9878d0886)


### 8. 重启主服务器

重启主服务器以使配置更改生效：

```bash
docker restart pgsmaster
```

### 9. 进入主服务器容器控制台

现在我们进入主服务器的容器：

```bash
docker exec -it pgsmaster /bin/bash
```

### 10. 创建复制角色

在从属服务器上，我们需要创建一个角色来处理复制：

```bash
psql -U postgres
# 关闭同步提交，从属服务器不会等待主服务器确认数据已写入后再提交事务。
set synchronous_commit = off;
# 创建复制角色
create role repl login replication encrypted password 'repl';
# 查看角色
\du
\q
exit
```

![image-20241102200723595](https://github.com/user-attachments/assets/89d627c0-15c9-4dd0-949d-0f7337a77333)

我们看到repl角色，说明创建成功

### 11. 进入从属服务器

现在我们进入从属服务器的容器：

```bash
docker exec -it pgsslave /bin/bash
```

### 12. 数据备份

使用 `pg_basebackup` 从主服务器备份数据：
此次备份是将整个数据库文件从主服务器中备份下来，而不是通过流的形式备份

```bash
pg_basebackup -Fp --progress -D /home/opt/postgresql-16.0/data/ -R -h <主服务器IP> -p 5432 -U repl --password
```

输入密码：repl

![image-20241102200723595](https://github.com/user-attachments/assets/b0ae5208-2447-4994-9a00-b12ea0bc5f27)

![image](https://github.com/user-attachments/assets/f33e479a-c5c3-4d10-8b32-78670263da16)


exit退出

### 13. 复制数据到宿主机

将从属服务器的数据复制到宿主机，以便我们可以在新的从属服务器中使用：

```bash
docker cp pgsslave:/home/opt/postgresql-16.0/data/ /var/lib/pgsslave
```

![image-20241102201244640](https://github.com/user-attachments/assets/a5d3eee3-52da-49ff-ad9c-5880c6fc8e56)


我们查看/var/lib/pgsslave包含完整的数据库文件

![image-20241102201552947](https://github.com/user-attachments/assets/8c9b5baf-936a-469b-a581-9307b9300bf6)


### 14. 删除从属服务器

删除旧的从属服务器容器：

```bash
docker rm -f pgsslave
```

### 15. 使用复制的数据创建新的从属服务器

重新创建从属服务器，使用之前备份的数据：

```bash
docker run --network=pg-network --name pgsslave -p 5501:5432 -e POSTGRES_PASSWORD=123456 -v /var/lib/pgsslave:/var/lib/postgresql/data -d postgres:16.4
```

![image-20241102201626937](https://github.com/user-attachments/assets/a7a3e8d8-88c0-4d1c-afc0-d63874cb1fb2)


### 16. 查看从属服务器日志

最后，查看从属服务器的日志，以确保复制正常运行：

```bash
docker logs -f pgsslave
```

我们发现日志中包含“recovery”、“WAL”等字样

![image-20241102201745535](https://github.com/user-attachments/assets/6bb2d525-4a05-4967-aff1-9bf5b56ba5b6)


### 测试

我们连接两个数据库

![image-20241102202338489](https://github.com/user-attachments/assets/308b3f68-03ff-490a-aa49-69a6c6cc6565)

在主服务器上创建表并插入数据

![image-20241102202704087](https://github.com/user-attachments/assets/28c94a72-3c5f-4306-827e-582911e1f598)

![image-20241102202718980](https://github.com/user-attachments/assets/4209420d-014d-425d-bb5f-1bab35c8b2d1)


我们打开从属服务器发现数据同步了


![image-20241102202811812](https://github.com/user-attachments/assets/5756cc1a-0858-4863-807f-cb6bb5bb4de9)

我们想在从属服务器插入数据发现插入失败

![image-20241102202950052](https://github.com/user-attachments/assets/8be45eaa-cdbf-4c56-96b6-ec6cec4a9794)


查看主服务器的复制日志表发现复制记录

![image-20241102203354607](https://github.com/user-attachments/assets/8f0edfe2-e8df-476c-b662-484491093cb2)


### 总结

通过以上步骤，我们成功地在 Docker 中创建了 PostgreSQL 主从复制环境。主从复制不仅提高了数据的可靠性，还可以帮助我们在负载较高时进行负载均衡、实现读写分离增强数据库的吞吐量。