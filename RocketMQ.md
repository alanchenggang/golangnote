### Q1. Rocket如何保证消息队列系统的高可用性?

1. 主从复制：RocketMQ 支持主从复制模式，其中一个 Broker 作为主节点，负责处理消息的读写操作，而其他的 Broker 作为从节点，复制主节点的数据。这样可以确保即使主节点发生故障，从节点仍然可以继续提供服务，保证消息队列的高可用性。

2. 故障自动转移：当主节点出现故障时，RocketMQ 会自动进行主节点的切换和故障转移。通过选举机制和心跳检测，从节点可以自动接替主节点的角色，确保消息的正常传输和服务的可用性。

3. 数据冗余：RocketMQ 具有数据冗余的特性，通过将消息复制到多个 Broker 上，即使其中一个 Broker 发生故障或不可用，仍然可以从其他副本中获取消息，确保数据的可靠性和高可用性。

4. 高可用部署：RocketMQ 支持将 Broker 和 NameServer 进行集群部署，以实现高可用性。通过部署多个 Broker 实例和多个 NameServer 实例，可以分担负载并提供冗余，从而保证系统的可用性和容错性。

5. 快速故障检测和恢复：RocketMQ 内置了故障检测和恢复机制。当 Broker 或 NameServer 出现故障时，RocketMQ 会快速检测并进行相应的故障恢复操作，以尽快恢复系统的可用状态。

6. 监控和报警：RocketMQ 提供了监控和报警机制，可以实时监控 Broker 和 NameServer 的运行状态、负载情况等关键指标。通过监控和报警系统，可以及时发现和解决潜在的问题，确保系统的稳定性和可用性。

---

### Q2. RocketMQ 如何避免消息的重复消费?

1. 消息消费的幂等性设计：应用程序在消费消息时，需要设计幂等性逻辑，确保同一条消息被重复消费时不会产生副作用。通过在应用程序中实现幂等性，即使消息重复消费，也能保证最终的结果一致。
2. 消息消费状态记录：RocketMQ 提供了消息消费进度的记录和管理机制。消费者可以通过将消费进度保存到持久化存储中（如数据库）来追踪已消费的消息。这样，当消费者重启后，可以从上次的消费进度继续消费，避免重复消费。
3. 消息消费确认机制：RocketMQ 提供了消息消费确认机制，消费者在消费完一条消息后，可以向 RocketMQ 发送消费确认消息，标识该消息已成功消费。如果消费者在处理消息过程中发生异常，没有发送消费确认消息，RocketMQ 会将该消息重新投递给其他消费者进行消费，确保消息不会丢失。
4. 消息消费超时设置：在消息消费过程中，可以设置合适的消费超时时间。如果消费者在超时时间内未发送消费确认消息，RocketMQ 将认为消费失败，并将消息重新投递给其他消费者进行消费。
5. 消息重复检测：RocketMQ 提供了消息重复检测机制，可以根据消息的唯一标识进行重复消息的检测和过滤。消费者可以通过自定义消息的唯一标识，并在消费前检查是否已经消费过该消息，避免重复消费。

---

### Q3. RocketMQ 如何避免消息的丢失?

1. 持久化存储：RocketMQ 使用可配置的持久化存储来存储消息。在消息到达 Broker 后，RocketMQ 会将消息持久化到磁盘上，以确保即使在出现故障或重启后，消息仍然可用。默认情况下，RocketMQ 使用 CommitLog 存储消息，可以选择使用文件存储或者 RockDB 存储。

2. 复制机制：RocketMQ 支持主从复制模式，将消息复制到多个 Broker。当消息发送到主节点后，RocketMQ 会将消息同步复制到从节点上，确保即使主节点发生故障，从节点仍然具有消息的副本，避免消息的丢失。

3. 刷盘机制：RocketMQ 提供了刷盘机制来确保消息的持久化。刷盘是指将内存中的消息写入磁盘，确保消息的持久化存储。RocketMQ 提供了异步刷盘和同步刷盘两种模式，可以根据需要进行配置。异步刷盘能够提高消息的发送性能，而同步刷盘能够提供更高的消息可靠性。

4. 数据冗余：RocketMQ 通过将消息复制到多个 Broker 上，实现数据的冗余存储。即使其中一个 Broker 发生故障，仍然可以从其他副本中获取消息，避免消息的丢失。

5. 消息确认机制：RocketMQ 采用消息确认机制来确保消息的可靠传输。在消息发送后，生产者会等待 Broker 的确认消息，确保消息已经成功存储到 Broker 上。只有在收到确认消息后，生产者才会认为消息已经被成功发送，避免消息的丢失。

6. 持久化索引：RocketMQ 使用持久化索引来加速消息的检索。通过索引，RocketMQ 可以快速定位消息在 CommitLog 中的位置，避免消息的丢失和检索延迟。

---

### Q4. RocketMQ 如何保证消息的按序到达?

1. 顺序消息发送：在发送消息时，可以指定消息发送的顺序。通过指定相同的消息队列选择策略（Message Queue Selector）和消息队列分区（Message Queue Partition），确保相同的业务标识（如订单号）被发送到同一个队列，从而保证消息的有序性。
2. 顺序消息消费：在消费消息时，可以使用顺序消费的模式。RocketMQ 提供了顺序消费模式，保证同一个消息队列中的消息按照发送顺序被消费。消费者可以通过设置消费者实例（Consumer Instance）的并发度为1，或者使用顺序消费的消息监听器接口（MessageListenerOrderly）来保证消息的有序消费。
3. 顺序消息队列：RocketMQ 的消息队列（Message Queue）在创建时可以设置顺序属性，保证同一个消息队列中的消息按照发送顺序存储和消费。消息队列的顺序属性可以通过设置消息队列的属性（Message Queue Property）来指定。
4. 顺序消息索引：RocketMQ 使用顺序消息索引机制来加速顺序消息的查找。索引可以根据业务标识（如订单号）等关键字段建立，以提高消息的查找效率和按序到达的速度。

需要注意的是，要确保消息的严格有序性，需要在发送方和消费方都进行相应的配置和逻辑设计。顺序消息的有序性是相对的，只在同一个消息队列范围内保证有序，而不是全局有序。

---

### Q5. RocketMQ 如何实现消息间的事务性?

1. 发送事务消息：生产者发送事务消息时，将消息发送到 Broker，但消息状态标记为 "待确认"（Half Message）。这个阶段相当于发送了一个预备消息。
2. 执行本地事务：生产者在发送事务消息后，需要执行本地事务来处理与该消息相关的业务逻辑。本地事务可以是数据库操作、文件操作、远程调用等任何与业务相关的操作。
3. 提交或回滚事务：本地事务执行完毕后，生产者需要根据事务的结果决定是提交该消息（Commit）还是回滚该消息（Rollback）。如果本地事务执行成功，生产者将消息状态更新为 "已确认"（Commit Message），表示该消息可被消费。如果本地事务执行失败或发生异常，生产者将消息状态更新为 "已回滚"（Rollback Message），表示该消息将被丢弃，不会被消费。
4. 消费确认：消费者接收到已确认的消息后进行消费，并向 Broker 发送消息消费确认（Consume Message）。
5. 定时任务处理：Broker 在接收到消息的提交或回滚确认后，会根据消息的状态进行相应的处理。如果消息被确认为提交，Broker 将该消息标记为可被消费状态；如果消息被确认为回滚，Broker 将该消息丢弃。

通过以上步骤，RocketMQ 实现了事务消息的机制，确保了消息在发送和消费过程中的一致性。如果事务消息在本地事务执行期间发生异常或超时，RocketMQ 提供了补偿机制，通过定时任务进行消息状态的检查和处理，以保证消息的最终一致性。

---

# RocketMQ路由中心NameServer

---
![img](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/img2843224-a7bdf308a8d3739b.png)
> NameServer 是 RocketMQ 消息队列的状态服务器（**服务发现功能**），集群中的各个服务都需要通过 NameServer 来了解集群中各个服务的状态。
> NameServer 中维护着 Producer 集群、Broker 集群、 Consumer 集群的服务状态。通过定时发送心跳数据包进行维护更新各个服务的状态。
> NameServer 可以部署多个，多个 NameServer 互相独立，不会交换消息。Producer、Broker、Consumer 启动的时候都需要指定多个 NameServer，各个服务的信息会同时注册到多个 NameServer 上，从而能到达高可用。



## NameServer启动流程

1. 解析配置文件,填充配置类对象NameServerConfig/NettyServerConfig 属性值

   ```java
           namesrvConfig = new NamesrvConfig();
           nettyServerConfig = new NettyServerConfig();
           nettyClientConfig = new NettyClientConfig();
           nettyServerConfig.setListenPort(9876);
           if (commandLine.hasOption('c')) {
               String file = commandLine.getOptionValue('c');
               if (file != null) {
                   InputStream in = new BufferedInputStream(Files.newInputStream(Paths.get(file)));
                   properties = new Properties();
                   properties.load(in);
                   MixAll.properties2Object(properties, namesrvConfig);
                   MixAll.properties2Object(properties, nettyServerConfig);
                   MixAll.properties2Object(properties, nettyClientConfig);
                   if (namesrvConfig.isEnableControllerInNamesrv()) {
                       controllerConfig = new ControllerConfig();
                       MixAll.properties2Object(properties, controllerConfig);
                   }
                   namesrvConfig.setConfigStorePath(file);
   
                   System.out.printf("load config properties file OK, %s%n", file);
                   in.close();
               }
           }
   
           MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);
   ```

   NameServerConfig 属性:

   ![image-20230227143256140](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230227143256140.png)

   

2. 根据启动属性创建NamesrvController实例,并初始化该实例(核心控制器)

   ![image-20230227143919497](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230227143919497.png)

   3. 注册 JVM 狗子函数并启动服务器,以便监听Broker/消息生成者的网络请求

## NameServer的路由元信息小结

![image-20230224210105068](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230224210105068.png)

### topicQueueTable Topic 消息队列路由信息，消息发送时根据路由表进行负载均衡

```java
    private final Map<String/* topic */, Map<String, QueueData>> topicQueueTable;
```

对于<code>QueueData</code> 

```java
public class QueueData implements Comparable<QueueData> {
    private String brokerName; 
    private int readQueueNums;
    private int writeQueueNums;
    private int perm;// 读写权限
    private int topicSysFlag; 
}
```

### brokerAddrTable  Broker 基础信息   包含 brokerName 、 所属集群名称 、 主备 Broker地址 

```java
    private final Map<String/* brokerName */, BrokerData> brokerAddrTable;
```

对于 <code>BrokerData</code>

```java
public class BrokerData implements Comparable<BrokerData> {
    private String cluster;
    private String brokerName;

    /**
     存储当前代理复制集群的所有单个实例的容器。 key是brokerId，value是单个broker实例的地址
     */
    private HashMap<Long /* brokerID */, String> brokerAddrs;
    private String zoneName;
    private final Random random = new Random();
    /**
     * Enable acting master or not, used for old version HA adaption,
     */
    private boolean enableActingMaster = false;
}

```

### clusterAddrTable Broker 集群信息存储集群中所有 Broker 名称 

```java
    private final Map<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
```

### brokerLiveTable Broker 状态信息   NameServer 每次收到心跳包时会替换该信息

```java
private final Map<BrokerAddrInfo/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
```

对于 <code>BrokerLiveInfo</code>

```java
class BrokerLiveInfo {
    private long lastUpdateTimestamp;
    private long heartbeatTimeoutMillis;
    private DataVersion dataVersion;
    private Channel channel;
    private String haServerAddr;
}
```

### filterServerTable  Broker 上的 FilterServer 列表，用于类模式消息过滤

```java
    private final Map<BrokerAddrInfo/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

## NameServer 路由注册流程

RocketMQ路由注册是通过Broker和NameServer的心跳功能实现的。Broker启动时向集群中所有NameServer发送心跳语句，每隔30s向集群中所有的NameServer发送心跳包。

## NameServer 路由删除流程

## NameServer 路由发现流程

# RocketMQ消息发送



---

## 消息结构

## Producer 启动流程

## 消息发送流程

## 批量消息发送

### RocketMQ如何避免消息重复消费

---

RocketMQ 通过消费者端维护消息消费的偏移量（offset）来避免消息重复消费的问题。

每个消费者在消费消息时，会将其消费的最新偏移量提交到 RocketMQ 服务器端。RocketMQ 会将这些偏移量保存下来，并在下次消费时将消息的偏移量与保存的偏移量进行比对，如果消息的偏移量大于保存的偏移量，则表示该消息还未被消费过，可以进行消费。如果消息的偏移量小于或等于保存的偏移量，则表示该消息已经被消费过，需要避免重复消费。

>
>RocketMQ 还提供了多种消息消费模式，可以根据业务需求选择不同的消费模式来避免消息重复消费的问题。比如，顺序消费模式（Orderly Consumption Mode）可以保证消息的顺序性，并且避免消息的重复消费；广播消费模式（Broadcasting Consumption Mode）可以让多个消费者同时消费同一批消息，但每个消息只会被消费一次，避免消息的重复消费。

### RocketMQ 如何保证消息按序到达

---

RocketMQ 通过分片（Sharding）和顺序消息的设计来保证消息按序到达。具体来说，RocketMQ 将一个 Topic 中的所有消息按照顺序分配到多个分片（Shard）中，每个分片只负责处理其中一部分消息，这样可以提高消息的并发处理能力。同时，每个分片内部保持消息的严格顺序，保证消息按照生产者发送的顺序进行投递。因此，在消费者端进行消费时，需要保证消费者能够按照相同的顺序从每个分片中消费消息，才能保证整个 Topic 中消息的顺序性。

>为了保证消息按照相同的顺序从每个分片中消费，RocketMQ 提供了两种顺序消息的消费模式：
>
>1. 顺序消费模式（Orderly Consumption Mode）：该模式下，一个消费者只会从一个分片中消费消息，消费过程中不会同时处理来自其他分片的消息。这种方式保证了消费者能够按照顺序消费每个分片中的消息，从而保证整个 Topic 中消息的顺序性。
>2. 广播消费模式（Broadcasting Consumption Mode）：该模式下，多个消费者同时消费同一批消息，但每个消息只会被消费一次，避免消息的重复消费。这种方式适合于需要多个消费者同时处理相同消息的场景，但不保证消息的顺序性。
