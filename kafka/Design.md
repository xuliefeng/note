# kafka 架构设计（笔记）

## 存储模块

### 高可用的保障设计/日志

1、主题 -> 分区 -> 每个分区都有一个 leader 节点和 n 个 follower 节点

- topic 主题在创建的时候指定分区数量和副本数，每个分区都是一个物理的存储目录。所有的分区副本数据应该都是一致的，kafka 允许不同的副本之间有延迟的存在

- 数据的存储文件
  xxx.index：索引文件
  xxx.log：数据存储文件
  xxx.timeindex：时间索引
  leader-epoch-checkpoint：

偏移量索引中每个索引项包含 2 个值：相对偏移量，该值保存的是针对当前偏移索引文件的起始索引的偏移量；物理地址，该值存储的是日志分段文件中对于的物理位置
时间索引中每个索引项包含 2 个值：时间戳，该值保存的是当前日志分段存储时的时间戳；偏移量地址，该值存储的是该时间戳对应的相对偏移量

一般情况下，日志文件的起始偏移量 logStartOffset 等于第一个日志分段的 baseOffset，但这并不是绝对的，logStartOffset 的值可以通过 DeleteRecordsRequest 请求（比如使用 KafkaAdminClient 的 deleteRecords（）方法、使用 kafkadeleterecords.sh 脚本）、日志的清理和截断等操作进行修改

日志的删除：删除策略有 3 个

1. 日期删除：日期删除支持多个时间单位，秒，分，时等。优先级最高的时秒级，并且删除日志分段优先取该分段日志的时间索引文件。如果索引文件中的不存在记录信息则取分段日志的修改时间来进行判断
2. 文件大小：设置超过需要删除的总日志分段文件大小，超过舍得大小的则被删除
3. 日志分段起始偏移量：基于日志起始偏移量的保留策略的判断依据是某日志分段的下一个日志分段的起始偏移量 baseOffset 是否小于等于 logStartOffset，若是，则可以删除此日志分段

如果所有的日志都过期了会先创建一个有效的日志分段，用来服务当前的业务。并且在删除时会在 Log 对象维护跳跃表中删除需要被移除的日志分段信息，保证没有其它线程对其进行读取操作。然后被删除的日志分段会被加上.deleted 后缀，然后交由专门删除日志文件的延迟任务来进行删除

对于删除策略以外，kafka 还提供了重复消息压缩的支持。对于相同 key 的消息仅保留最新的消息，压缩完成之后的消息的消息偏移量跟之前的写入偏移量一致（新的日志分段文件）。由于有删除的操作压缩完的新日志分段文件中的偏移量是分连续的但是不影响数据的查询

### 磁盘存储

kafka 的文件存储使用硬盘来作为存储载体，这是因为在使用硬盘的顺序写盘时的速度的吞吐量也是不可小窥的。这是 kafka 考虑使用硬盘存储的因素之一

页缓存：把一部分内存当作硬盘的缓存，这样一来就可以实现对硬盘的操作变成了对内存对操作。使用文件系统并依赖于页缓存的做法可以省去一份进程的缓存开销，同时还可以使用结构紧凑的字节码来代替对象的使用方式以节省更多的内存和 gc 带来的性能损失。并且在服务重启之后页缓存还有效，而进程内存却需要重建。页缓存的同步是由操作系统进行控制，但是 kafka 提供了强制刷新的选择，不过数据的可靠性应该有多副本机制来保障而不是由强制刷新来控制

零拷贝：零拷贝是指将数据直接从磁盘文件复制到网卡设备中，而不需要经由应用程序之手。零拷贝大大提高了应用程序的性能，减少了内核和用户模式之间的上下文切换（java FileChannal.transferTo）

- 零拷贝技术通过 DMA（DirectMemoryAccess）技术将文件内容复制到内核模式下的 ReadBuffer 中。不过没有数据被复制到 SocketBuffer，相反只有包含数据的位置和长度的信息的文件描述符被加到 SocketBuffer 中。DMA 引擎直接将数据从内核模式中传递到网卡设备（协议引擎）。这里数据只经历了 2 次复制就从磁盘中传送出去了，并且上下文切换也变成了 2 次。零拷贝是针对内核模式而言的，数据在内核模式下实现了零拷贝。

### 生产者

kafka 的消息发送经过多个步骤才会真实的发送到 broker 上。 start -> 拦截器 -> 序列化器 -> 分区器 -> 消息累加池 -> 异步线程发送；发送线程会将 分区 转化为 Node 信息，消息对象转化为协议交互对象然后在发送出去

#### 参数

1. acks：消息发送后，判断成功选择。1 只需要 leader 分区副本接收成功即可，存在消息丢失风险（leader 接收后下线）；0 无须任何成功相应；-1 需要所有的 ISR 副本相应接收成功，但是对性能影响最大
2. retries 和 retry.backoff.ms：重试次数和重试间隔时间，配置成允许重试情况下，kafka 内部发送遇到可重试类型异常是内部可以进行重试发送而不是一位的抛出异常信息。而重试间隔时间可以规避一些瞬间服务异常的无意义重试
3. max.in.flight.requests.per.connection：生产者线程在单个 Socket 连接上能够发送未应答请求的最大数量（默认 5），对于强制消息有序性的业务该参数建议设置成 1，而非同个 acks 0 的模式来进行顺序控制
4. linger.ms：指定生产者发送 ProductBatch 消息前等待更多消息加入 ProductBatch 的时间（默认 0）。生产者会在 ProductBatch 被填满或者等待超时时间超过 linger.ms 值是发送出去

### 消费者

kafka 的消息者归属于一个消费者组，不同的组之间消费者是隔离的。同一个消息只会被一个消费者组中的一个消费者消费

kafka 的消费者进行消息的订阅有多种方式，subscribe 和 assign 两大类。subscribe 类型通过 topic 名称进行订阅注册（支持正则）不支持自定义分区选择，所以是由 kafka 自动管理分区的分配信息，在有新的分区或分下线时内部会自动进行更新；assign 方式可以自定义需要消费的分区，不支持自动分区切换。

订阅主题分区分配策略有多种，自定义选择需要的策略类型，支持多选通过逗号分割

1. RangeAssignor：n = 单个主题分区总数 / 消费者总数（同个消费组），m = 单个主题分区总数 % 消费者总数（同个消费组）。每个消费者分配 n 个分区跨度（相连的分区），多出来的 m 个分区按照消费者名字典排序从前完后分配
2. RoundRobinAssignor：所有主题的分区按照消费者（同个消费族）轮训进行分配。该分配模式下如果消费者订阅的主题不一样则不会分配到对应的分区信息，则出现多个消费者空闲，某个消费者繁忙（订阅的主题比其它的消费者多）
3. StickyAssignor：a) 分区的分配尽可能均匀；b）分区的分配尽可能与上次分配保持相同；两者冲突 a）优先级更高。尽可能的保持一致在出现消费者下线时，其它消费这分配的原有分区可能不会变动只是将已下线的订阅分区从新分配到每个消费者上面，这样就减少了不必要的性能浪费（如果都从新分配一次则可能会出现消息会在消费一遍）

每个消费者可以定义不同的分配策略，彼此的分配需要进行协调。对此 kafka 通过消费者协议器和组协调器来完成的。对于消费者所有的组进行分割，每个子集对应一个服务器端 GroupCoordinator 管理器，该 GroupCoordinator 负责消费组的管理组件，消费者客户端中的 ConsumerCoordinator 组件负责与之通信。

服务端 GroupCoordinator 会根据每个消费者的分区策略进行选择，找出每个消费者都支持的分区策略。然后通过消费者 leader 进行消费者的具体分区分配，最终然后通过服务端 GroupCoordinator 进行分配的通知

## 高效延迟任务

kafka 的延迟任务通过时间轮 + DelayQueue 组合实现，每个时间轮的槽对应一个过期任务的集合列表。对于当前时间轮的时间无法满足的任务会上溢到更高的层级时间轮，每个时间轮的最小时间作为 DelayQueue 的过期时间。每次过期时间推进的操作只需要从 DelayQueue 中获取最早的任务列，如果是高层级的时间轮队列里面的每个任务的时间是不一致的全部在重新执行一次加入时间轮操作，由于添加操作对于已经过期的任务会返回执行失败状态，对于添加失败的任务默认就会进行任务执行。所以此时已经过期的任务就会立刻执行轮而时间还未到期的则会从新加入到时间轮中，此时的时间对比上次加入的时间是变动的所以加入的时间轮层次可能就会进行降级直到一直到最第级（最精准层级）

重点：DelayQueue 的过期时间是 每个时间轮槽的最小时间

## 控制器

一个 kafka 集群中有且只有一个控制器节点（控制器节点通过 zk 来进行竞争，每个控制器都有一个对于的控制纪元），担任控制器的 broker 的节点会比起其它的普通 broker 多一份职责

1. 监听分区的变化
2. 监听主题的变化
3. 监听 broker 的变化
4. 从 zookeeper 中读取当前所有的主题、分区以及 broker 以及 broker 相关的信息进行管理
5. 启动并管理分区状态机和副本状态器
6. 更新集群的元数据
7. 维护分区的优先副本的均衡（可选）

控制器在对于所有的处理时 使用单线程 + 队列方式进行处理。不使用锁机制主要是基于性能考虑

集群中的 leader 副本选举也是通过控制器来进行，在出现 leader 下线（创建分区）与分区重分配是都会触发 leader 的从新选举。

1. leader 下线触发的选举使用的策略是 OfflinePartitionLeaderElectionStrategy。该策略是从 AR 集合中副本的顺序查找第一个存活的副本，并且这个副本在 ISR 集合中。并且设置允许从非 ISR 列表中选举 leader，则在不存在 ISR 是从 AR 中找到第一个存活副本设置为 leader
2. 分区重分配使用的策略是 ReassignPartitionLeaderElectionStrategy。该策略是在重分配的 AR 集合中找到第一个存活的副本，且这个副本是在目前的 ISR 列表中
3. 优先副本选举，直接将优先副本设置为 leader，AR 集合中的第一个副本就是优先副本
4. 当某个节点在被优雅的下线时，该节点上的 leader 副本都会下线，所以对应的分区都要执行 leader 的选举。此时从 AR 列表中找到第一个的存活的副本，且这个副本在目前的 ISR 列表中，且不再被关闭的节点上

## 幂等性/事务

kafka 中的生产者通过消息的幂等性来控制消息的重复发送问题，消息的丢失通过 acks 机制进行控制。由于消息的幂等操作是通过生产者 ID + 分区 ID 来实现的，所以幂等操作只对同个分区有效

事务主要针对生产者来使用，每个生产者需要显示的设置一个事务 ID，kafka 通过事务 ID + 生产者 ID + 生产者周期（epoch） ID 来确认唯一标示。开启事务 ID 默认开启幂等性功能操作，事务性消息提交要么全部成功要么全部失败（跨分区消息）。对于消费者而言事务消息最主要的影响是能否消费未提交的事务消息

## 可靠性

### 副本

每个消息的分区都有 leader 副本和 follower 副本，而 follower 副本又分为两种角色；ISR 副本和失效副本，follower 副本的失效判断是通过当前的时间和该 follower 副本最后一次数据拉取到 leader 副本的 LEO 消息时的时间来判断是否超过一个设置的时间值判断，超过该时间的副本则认为是无效副本等到下次追赶上 HW 是从新加入 ISR 副本集中。

### 不支持读写分离

1. 数据一致性问题。数据从主节点转到从节点必然会有一个延时的时间窗口，这个时间窗口会导致主从节点之间的数据不一致。某一时刻，在主节点和从节点中 A 数据的值都为 X，之后将主节点中 A 的值修改为 Y，那么在这个变更通知到从节点之前，应用读取从节点中的 A 数据的值并不为最新的 Y，由此便产生了数据不一致的问题。
2. 延时问题。类似 Redis 这种组件，数据从写入主节点到同步至从节点中的过程需要经历网络 → 主节点内存 → 网络 → 从节点内存这几个阶段，整个过程会耗费一定的时间。而在 Kafka 中，主从同步会比 Redis 更加耗时，它需要经历网络 → 主节点内存 → 主节点磁盘 → 网络 → 从节点内存 → 从节点磁盘这几个阶段。对延时敏感的应用而言，主写从读的功能并不太适用。

kafka 通过副本分区的机制将不同的分区 leader 分布在不同的 broker 上，这样一来也就实现了业务的负载均衡。并且由 leader 副本进行读写处理还保证了数据同步的有序性

## 命令工具

1、创建 topic
./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

2、生产者命令
./kafka-console-producer.sh --broker-list localhost:9092 --topic test

3、消费者命令
./kafka-console-cusumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
