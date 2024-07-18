## Docker服务器上部署最新版Kafka
### 博客地址：https://blog.tmdog114514.icu

### 前提条件
在开始之前，请确保你已经安装了以下环境：
1. Docker

### 创建目录
首先，我们需要创建一个目录来存储Kafka的相关数据：
```bash
mkdir -p /data/deploy/kafkaCluster/kraft
```

### 创建docker-compose.yaml文件
在你自己的目录下创建一个名为`docker-compose.yaml`的文件，并添加以下内容：
我是在root目录下创建了kafka_config文件夹
![image](https://github.com/user-attachments/assets/08c20808-60b2-44a8-b3c6-175b74d0e1f7)

在docker-compose.yaml下：

```yaml
version: "3"
services:
   kafka:
     image: 'bitnami/kafka:latest'
     user: root
     environment:
       - KAFKA_ENABLE_KRAFT=yes
       - KAFKA_CFG_PROCESS_ROLES=broker,controller
       - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
       - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
       - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
       - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://ip:9092
       - KAFKA_BROKER_ID=1
       - KAFKA_CFG_NODE_ID=1
       - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@localhost:9093
       - ALLOW_PLAINTEXT_LISTENER=yes
     volumes:
       - /data/deploy/kafkaCluster/kraft:/bitnami/kafka:rw
     ports:
       - "9092:9092"
       - "9093:9093"
```

### 配置解释
- `version: "3"`：指定Docker Compose文件的版本。
- `services`：定义服务，本例中为Kafka服务。
- `image: 'bitnami/kafka:latest'`：使用Bitnami提供的Kafka最新镜像。
- `user: root`：以root用户运行容器。
- `environment`：设置环境变量，配置Kafka。
  - `KAFKA_ENABLE_KRAFT=yes`：启用KRaft模式。
  - `KAFKA_CFG_PROCESS_ROLES=broker,controller`：设置Kafka的角色为broker和controller。
  - `KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER`：定义controller的监听器名称。
  - `KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093`：定义Kafka的监听地址。
  - `KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT`：映射安全协议。
  - `KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://ip:9092`：设置广告监听地址，请将`ip`替换为服务器的实际IP地址。
  - `KAFKA_BROKER_ID=1`：设置Broker ID。
  - `KAFKA_CFG_NODE_ID=1`：设置节点ID。
  - `KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@localhost:9093`：设置控制器选举的投票者。
  - `ALLOW_PLAINTEXT_LISTENER=yes`：允许PLAINTEXT监听。
- `volumes`：挂载主机目录到容器内，持久化数据。
  - `/data/deploy/kafkaCluster/kraft:/bitnami/kafka:rw`：将主机的`/data/deploy/kafkaCluster/kraft`目录挂载到容器的`/bitnami/kafka`目录。
- `ports`：暴露端口。
  - `"9092:9092"`：将主机的9092端口映射到容器的9092端口。
  - `"9093:9093"`：将主机的9093端口映射到容器的9093端口。

可以通过添加下面的配置指定Kafka 控制器集群中的仲裁投票者（Quorum Voters）,目前配置是单机所以没有去配置
```yaml
KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=brokerId@host:port,brokerId@host:port,...
```

### 部署Kafka
在运行docker-compose的目录下，运行以下命令启动Kafka服务：
```bash
docker-compose up -d
```

结果如下：
![image](https://github.com/user-attachments/assets/9bcd046f-f75c-4dce-bdee-029e13a39486)


### 测试部署
确认Kafka服务是否已成功运行，可以使用以下命令检查容器状态：
```bash
docker ps
```
结果如下：
![image](https://github.com/user-attachments/assets/0c6e6de2-df26-4aa2-908b-f65231be3a8b)

如果Kafka容器正在运行，你将看到`bitnami/kafka:latest`镜像的容器在列表中。

你还可以使用Kafka命令行工具或Kafka客户端测试Kafka服务，例如使用Kafka自带的生产者和消费者工具发送和接收消息。

---

这样，我们就完成了在基于Ubuntu的Docker服务器上部署最新版Kafka的操作。希望这篇博客对你有所帮助！