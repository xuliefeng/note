# kafka 架构设计（笔记）

## 存储模块

### 高可用的保障设计

1、主题 -> 分区 -> 每个分区都有一个 leader 节点和 n 个 follower 节点

- topic 主题在创建的时候指定分区数量和副本数，每个分区都是一个物理的存储目录。所有的分区副本数据应该都是一致的，kafka 允许不同的副本之间有延迟的存在

- 数据的存储文件
xxx.index：索引文件
xxx.log：数据存储文件
xxx.timeindex：时间索引
leader-epoch-checkpoint：


## 命令工具

1、创建 topic
./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

2、生产者命令
./kafka-console-producer.sh --broker-list localhost:9092 --topic test

3、消费者命令
./kafka-console-cusumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning

