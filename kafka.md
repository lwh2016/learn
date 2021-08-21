## 一. 技术架构

### 1.zookeeper

> + kafka使用zookeeper保存集群的元数据信息和消费者信息（？？？，高版本的消费者管理是不是通过kafka自身的topic 来实现），zookeeper基于一致性实现，当群组中多个broker不可用时，整个集群就不可用
>
> + zookeeper节点之间通信的port有3个，clientport、peerport、leaderport，客户端通过clientport连接群组，peerport用于节点间通信的tcp端口，leaderport用于首领选举的TCP端口
>
> + zookeeper可以用于管理broker和消费者，进行broker配置，在消费者发生宕机时进行rebalance

### 2.kafka集群的技术框架

> 基于zookeeper和controller实现集群管理，基于topic实现数据管理，基于复制机制和多副本机制实现数据高可用性和可靠性，基于文件系统实现数据持久性

###  3.Zero-Copy机制  ？？？

> 零复制机制指的是kafka在发送消息时直接把消息从文件复制到网络通道，不需要通过任何缓冲区，这就跟大多数数据库系统不一样的地方，大多数的数据库系统在发送到客户端之前会先把他们保存到本地缓存里。zero-copy机制避免了字节复制，也不用管理内存缓冲区，从而获得更好的性能

### 4. at-least-once 

> + kafka能保证至少消费一次，但不能保证 only-once
> + 可以配合其他外部系统实现 only-once，有两种方案：
>   + 把数据写到支持唯一键的系统，比如es、mysql，来保证同一个键值的数据只会保留一个，可以用 topic+partition+offset 实现
>   + 借助于支持事务的系统，把消息处理和偏移量提交放在同一个事务里

## 二.消息传输机制

### 1.生产者

> + 向kafka集群发送消息的客户端，同一个生产者可以往多个topic发送消息
> + 生产者决定会把消息发送到kafka的哪个partition，分区规则由分区器实现，分区器可以自己实现也可以使用默认的，默认的分区策略是：如果key值是空，则按照顺序分到下一个分区，否则对key值进行hash分区
> + 生产者在收到错误时需要进行恰当的处理，比如收到 LeaderNotAvaliableException错误需要进行重发，否则会出现生产者没有正确处理导致的数据丢失，这是生产者自动处理的，不需要开发者手动处理；还有一类错误是不可重试错误，比如序列化错误、重试次数达到上限等，需要开发者自己处理
> + 可靠性配置：1）acks配置   2）重试次数配置   3）生产者对不可重试错误的处理方式
> + 生产者首先会把消息放到缓冲区，当消息积累到一定量时或者等待到达一定时长，会整理成batch统一发送出去；所以整个消息发送过程分成两部分：
>   + 把消息放到发送缓冲区
>   + 调用网络I/O发送消息到kafka集群
>   + 如果生产者生产数据过快导致缓冲区满了，可能也会出现异常 ？？？
>
> + 基于kafkaProducer 对象实现消息发送，需要配置一些 参数 ，每个记录成为一个producerRecord，调用send（）实现消息发送，发送到每个partition的消息会分batch发送出去
> + 调用send（）方法会先进行序列化，然后传给分区器，如果ProcuderRecord没有指定partition就会在这里处理；消息被发送到服务器端之前，会先被放到缓冲区里，然后使用单独的线程发送到服务器端

### 2.消费者

> + 从kafka获取消息的客户端，通常配置的是消费者组（Consumer Group），一个消费者组内由多个消费者
> + 每隔consumer只能从属于一个消费者组（consumerGroup），也可以不配置消费者组 ，如果不设置会默认配置还是怎样 ？？
> + 每个消费者只能消费一个topic的一个分区，具体消费哪个分区 由consumerGroup来分配 ？？
> + kafka的消费机制采用拉取机制，由消费者控制消费进度和消费速度，消费者可以从任意的偏移量进行消费
> + 没有同步到所有同步副本的消息是不会发送给消费者的，从而确保不同消费者获得消息的一致性
> + 具体消费者的偏移量提交到哪里？？？ 消费者组对消费者的管理包括哪些方面 ？？？
> + 如果消费者在没有处理完消息之前就提交了偏移量，这些消息永远不会被再次消费，即使换了新的消费者重新接手，这些消息也会被认为是已消费的，这也是数据丢失的主要原因
> + 所有消费者接收到的消息都是满足一致性的，这里的一致性指的是不同消费者组获取到的消息都是一致的
> + 轮询：用于请求信息、发送心跳、分区再均衡，consumer.poll(阻塞时间) 是轮询API，可以在轮循里提交偏移量  ？？？
> + 线程安全：每个消费者都应该占用一个线程，多个线程之间也不要共享消费者   ？？？
> + rebalance: 指的是消费者发生宕机时，重新设置每个消费者和topic对应分区数据的消费关系 ；再均衡通过群组协调器进行监听和协调
>
> + 一个topic的分区只能被一个消费者组里的消费者消费，如果消费者数量大于分区数量，会导致一些消费者闲置，不会接收任何消息

### 3.topic、partition、offset

> + topic是对消息的一种分类，kafka使用topic来组织数据，相当于数据库中的表，kafka中的每条消息都从属于一个topic 
>
> + partition是一种逻辑上的概念，partition是kafka存储的基本单元，用于对数据进行分区，实现分布式场景下的消息并发生产和消费，每个partition数据对应一个分区的日志文件（append log），为了分区文件不至于过大，每个分区的日志文件由多个segment构成，这样便于提高查询效率，segment文件只能追加不能修改
>
> + 分区中的每条消息都会分配到一个单调递增的消息序号，就是我们说的 offset，是个long型值，每个partition内部的消息严格按照时间顺序排列，按照先入先出的顺序进行消费，但是不同分区间的数据不能保证有序性

### 4.副本

> + 首领副本称为leader，每个partition都有一个leader，生产者和消费者的请求都会经过这个副本
>+ 首领副本可以根据跟随着副本请求的消息偏移量判断哪些副本是同步副本，只有同步副本才会成为新leader，跟随者副本负责从leader处复制消息，消息的复制机制跟消费者的消费机制是一样的，也是向leader申请读取某个偏移量的record
> + leader崩溃之后，其中的一个跟随者副本会成为新leader，就是跟随者副本列表中的第一个，这里有个问题：非同步副本是否会出现在列表中   ？？？ 
>+ 选举新leader时，controller会遍历所有的节点找出新的leader，并向新leader和跟随着发送请求，该请求里包含着谁是leader以及谁是分区跟随着的信息
> + 生产者和消费者的请求都必须发往leader所在的broker，客户端可以通过元数据请求获取leader信息及副本信息，元数据请求可以发往任意一个broker，因为每个broker都保存了这些元数据信息，客户端会把元数据信息缓存起来，时不时发送请求获取元数据信息，当间隔时长到metadata.max.age.ms配置的时长或者收到 “非首领”错误 响应时重新获取
> + 同步副本指的是和leader数据同步的副本，同时要求近期内和zookeeper有会话、和leader有数据通信；只有同步机制可以被选为新的leader；同步副本的数据也会影响性能，只有复制到所有同步副本的消息才会认为"已提交"；虽然非同步副本不在跟leader同步，但是我们不关系他的数据，因此不会影响性能

### 5.controller 

> + 每个kafka集群会选举出一个broker作为controller，controller用于选举副本中的首领副本、？？？
> + 每个controller都会由自己的编号（epoch），epoch是连续递增的，controller崩溃之后，其他broker会检测到controller节点消失，都会申请成为新的broker，第一个申请成功的会成为controller，其他broker收到“节点已存在”的消息
> + 其他broker在知道谁是新的controller之后，再收到 epoch编号较旧的controller的信息会忽略他们，使用epoch来避免“脑裂”

### 6.物理存储

> + 文件管理：文件存储是kafka的一个基本特性，kafka不会一直保存数据，每个主题都配置了数据保存时长，默认是7天。文件会在达到数据保存时长（log.retention.hours）或者文件大小达到限制时（log.segment.bytes）删除
>
>   注：这个时间是从文件关闭的时间开始算
>
> + 为了提高消息的读取速度，kafka维护索引文件来记录偏移量映射到segment文件以及偏移量在文件中的位置；索引文件跟segment对应，segment删除时索引文件也可以删除

### 7.broker

> + 影响broker可靠性的几个参数：复制系数、不完全首领选举机制和最小的同步副本个数，保证可靠性就会丧失高可用性或者磁盘成本

## 三. 配置及使用说明

### 1.broker 

> + unclean.leader.election.enable:是否允许不完全首领选举机制，即是否允许不同步副本成为leader，默认是true；如果允许，则可用性会变高，但是可靠性会降低
> + message.max.bytes:接收的消息的最大值

### 2.zookeeper

> 

### 3.topic配置

> + log.dir: 配置日志文件的存储位置，默认 /tmp/kafka-logs
> + replication.factor: 复制系数，即分区的副本数；broker里配置参数为 default.replication.factor ，默认的复制系数，默认是 3 ；复制系数为N，则最少需要N个broker，每个副本需要在不同的broker上；副本个数越多，集群可用性越高，但是占用的磁盘资源也会越多，因此需要在可用性和磁盘成本之间做出权衡
> + min.insync.replicas:最小的同步副本数量，broker也可以配置

###  4.生产者  

> + bootstrap.servers: kafka broker 列表配置，不必配置所有的集群ip，连接到一个kafka broker 之后会从 集群元数据中获取其他broker的信息
>
>   分布式场景下为了保证集群的可用性，保证一个broker宕机后仍旧能连接上集群，最少配置两个broker信息
>
> + acks:生产者的消息发送成功确认模式，包含 acks=1（leader接收就返回给生产者 "发送成功"的响应）、acks=all(所有同步副本都复制完成)、acks=0（通过网络发送出去就认为写入kafka成功，怎么算网络发送成功？？？）
>
> + key.serializer/value.serializer: key和value必须要进行配置，可以进行自定义
>
> + buffer.memory:设置生产者缓冲区的大小，生产者用它缓冲发送到服务器的消息
>
> + max.block.ms:调用send()或者partitionFor() 时生产者的最大阻塞时间，如果调用send 缓冲区已满，或者调用partitionFor()获取元数据失败，生产者就会阻塞，超过阻塞时间就会抛出生产者异常
>
> + reties：生产者遇到错误时的重试次数，可以用于解决首领异常、网络异常、缓存区已满等可重试异常
>
> + batch.size/linger.ms:发送到同一个parition的消息，生产者会把他们放到同一个batch里发送出去。这两个参数指定了batch的字节大小上限或者等待时间
>
> + max.in.flight.requests.per.connection:指定了生产者在收到服务器的消息确认前，允许发送多少个消息；如果对消息有严格次序限制，可以设置为1
>
> + timeout.ms:指定了leader在收到同步副本的确认前，最大的等待时长；如果在限制时间内没有收到确认，broker就会返回异常
>
> + request.timeout.ms:生产者在收到broker确认消息发送成功之前的最大等待时长
>
> + metadata.fetch.timeout.ms:生产者获取broker元数据时的最大等待时长
>
> + max.request.size:控制生产者发送的请求大小
>
> + receive.buffer.bytes/send.buffer.bytes:TCP socket 接收和发送数据包的缓冲区大小

###  5.消费者   

> + group.id: 消费者组 id，同一个消费者组会获取到全部的消息，但是内部的每个消费者只能看到一部分消息
> + auto.offset.reset:在偏移量无效时或者没有偏移量可提交时的偏移量设置策略，包括两种：earliest和latest
> + enable.auto.commit:自动提交配置，也可以在带代码里手动提交代码
> + auto.commit.interval.ms:自动提交的时间间隔
> + fetch.min.bytes/fetch.max.interval.ms:拉取消息是等待获取的最小的字节量或者最长等待时长
> + max.partition.fetch.bytes：从broker拉取的最大的字节长度，这个参数要配置的比 max.message.size 大，否则可能导致 消息者无法读取这些消息，从而一直挂起
> + max.poll.records:单次拉取的最大的记录条数
> + session.timeout.ms：协调器在没有收到消费者的心跳超过这个时长时会被认为已四方
> + heart.timeout.ms: 消费者每次向协调器发送心跳的频率

## 四.附录

1. 可以登录 https://kafka.apache.org/documentation/#api 查看kafka文档
2. 参考 《kafka权威指南》











