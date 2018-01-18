---
title: RocketMQ
date: 2017-06-02 23:30:24
tags: [RocketMQ, Java]
---

# 自建 RocketMQ

## RocketMQ
阿里开源的一款高性能、高吞吐量的消息中间件. 现在为Apache 孵化器项目

## 支持消息类型
- 有序
- 普通无序(现在我们的订阅模式都是无序模式)

<!-- more -->

## 负载均衡
- producer 轮询的方式进行负载均衡
- consumer
  - 遍历Consumer下的所有topic，然后根据topic订阅所有的消息
  - 获取同一topic和Consumer Group下的所有Consumer
  - 然后根据具体的分配策略来分配消费队列，分配的策略包含：平均分配、消费端配置等

```
                                         +consumer A
                    Topic A              |
                                         |
                +------------>  <--------+
                |                        |
                |   broker               |
                |                        |
producer   ------------------>  <--------+
                |
                |   broker
                |
                +------------>  <---------+
                                          +consumer B
                    broker
```

## 物理部署结构

```
               +----------------------+
               |                      |
               |     namesrv cluster  |
+-------------^+                      <------------+
|              +------+-----+---------+            |
|                     ^     ^                      |
|                     |     |                      |
|                     |     |                      |
|                     |     |                      |
|                     |     |                      |
|      +--------+     |     |      +-------+       |
|      |        +-----+     |      |       |       |
|      |        |           +------+       |       |
|      | master |                  | slave |       |
|      |        +------------------+       |       |
|      |   A    |                  |  A    |       |
|      |        |                  |       |       |
|      |        |                  |       |       |
|      +--------+                  +-------+       |
|                                                  |
|                                                  |
|                                                  |
|                                                  |
|      +--------+                  +-------+       |
|      |        |                  |       |       |
+------+        |                  |       +-------+
       | master |                  | slave |
       |        +------------------+       |
       |   B    |                  |   B   |
       |        |                  |       |
       |        |                  |       |
       |        |                  |       |
       +--------+                  +-------+
```


### 部署遇到的问题

```code
physic disk maybe full soon, so reclaim space, 0.9200628672685716
disk space will be full soon, but delete file failed.
```

看源码
```java
class CleanCommitLogService {

        private final static int MAX_MANUAL_DELETE_FILE_TIMES = 20;
        private final double diskSpaceWarningLevelRatio =
            Double.parseDouble(System.getProperty("rocketmq.broker.diskSpaceWarningLevelRatio", "0.90"));

        private final double diskSpaceCleanForciblyRatio =
            Double.parseDouble(System.getProperty("rocketmq.broker.diskSpaceCleanForciblyRatio", "0.85"));
        private long lastRedeleteTimestamp = 0;
        /* ignore */
}
```

这是因为, RocketMQ 的磁盘空间检测是根据比例来计算的, 所以, 剩余磁盘比例要求, 需要可用空间大于10%

### 跳过阿里的验证 ClientRPCHook (ONSClient)
```java
@Override
    public void doBeforeRequest(String remoteAddr, RemotingCommand request) {
        CommandCustomHeader header = request.readCustomHeader();
        // sort property
        SortedMap<String, String> map = new TreeMap<String, String>();
        map.put(SessionCredentials.AccessKey, sessionCredentials.getAccessKey());
        map.put(SessionCredentials.ONSChannelKey, sessionCredentials.getOnsChannel().toString());
        try {
            // add header properties
            if (null != header) {
                Field[] fields = fieldCache.get(header.getClass());
                if (null == fields) {
                    fields = header.getClass().getDeclaredFields();
                    for (Field field : fields) {
                        field.setAccessible(true);
                    }
                    Field[] tmp = fieldCache.putIfAbsent(header.getClass(), fields);
                    if (null != tmp) {
                        fields = tmp;
                    }
                }

                for (Field field : fields) {
                    Object value = field.get(header);
                    if (null != value) {
                        map.put(field.getName(), value.toString());
                    }
                }
            }
            byte[] total = AuthUtil.combineRequestContent(request, map);
            String signature = AuthUtil.calSignature(total, sessionCredentials.getSecretKey());
            request.addExtField(SessionCredentials.Signature, signature);
            request.addExtField(SessionCredentials.AccessKey, sessionCredentials.getAccessKey());
            request.addExtField(SessionCredentials.ONSChannelKey, sessionCredentials.getOnsChannel().toString());
        }
        catch (Exception e) {
            throw new RuntimeException("incompatible exception.", e);
        }
    }
```

ONS的SDK里的实现是他重新实现自己的生产者与消费者, 增加ONS的RPCHook, 在每次的请求加上ONS的校验信息, 我们自搭建的不进行这些校验, 所以, 请求都会通过.

### broker 参数

#### 使用broker配置模板
导出配置文件
```
sh mqbroker -m > anyfile
```

```
namesrvAddr=192.168.1.50:9876
brokerIP1=192.168.1.50
brokerName=localhost.localdomain
brokerClusterName=DefaultCluster
brokerId=0
autoCreateTopicEnable=true
autoCreateSubscriptionGroup=true
rejectTransactionMessage=false
fetchNamesrvAddrByAddressServer=false
storePathRootDir=/root/store
storePathCommitLog=/root/store/commitlog
flushIntervalCommitLog=1000
flushCommitLogTimed=false
deleteWhen=04
fileReservedTime=72
maxTransferBytesOnMessageInMemory=262144
maxTransferCountOnMessageInMemory=32
maxTransferBytesOnMessageInDisk=65536
maxTransferCountOnMessageInDisk=8
accessMessageInMemoryMaxRatio=40
messageIndexEnable=true
messageIndexSafe=false
haMasterAddress=
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
cleanFileForciblyEnable=true
```

字段|默认|说明
---|--|--
brokerRole | ASYNC_MASTER | Broker 的角色 - ASYNC_MASTER 异步复制 Master - SYNC_MASTER 同步双写 Master - SLAVE
flushDiskType | ASYNC_FLUSH | 刷盘方式 - ASYNC_FLUSH 异步刷盘 - SYNC_FLUSH 同步刷盘 <b>ASYNC_FLUSH is recommended, for SYNC_FLUSH is expensive and will cause too much performance loss. If you want reliability, we recommend you use SYNC_MASTER with SLAVE.
cleanFileForciblyEnable | TRUE | 磁盘满、且无过期文件情况下 TRUE 表示强制删除文件，优 先保证服务可用 FALSE 标记服务不可用，文件 不删除

### 官方推荐broker集群搭建方式(这里的 Slave 不可写，但可读，类似于 Mysql 主备方式)

0. <s>单master
    > 这种方式风险较大，一旦 Broker 重启或者宕机时，会导致整个服务不可用，不建议线上环境使用

0. <s>多master
    > 一个集群无 Slave，全是 Master，例如 2 个 Master 或者 3 个 Master
    > 优点：配置简单，单个 Master 宕机或重启维护对应用无影响，在磁盘配置为 RAID10 时，即使机器宕机不可恢复情况下，由于 RAID10 磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢）。性能最高。
    > 缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到受到影响。

0. 多 Master 多 Slave 模式，异步复制
    > 每个 Master 配置一个 Slave，有多对 Master-Slave，HA 采用异步复制方式，主备有短暂消息延迟，毫秒级。
    > 优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，因为 Master 宕机后，消费者仍然可以从 Slave 消费，此过程对应用透明。不需要人工干预。性能同多 Master 模式几乎一样。
    > 缺点：Master 宕机，磁盘损坏情况，会丢失少量消息。

0. 多 Master 多 Slave 模式，同步双写
    > 每个 Master 配置一个 Slave，有多对 Master-Slave，HA 采用同步双写方式，主备都写成功，向应用返回成功。
    > 优点：数据与服务都无单点，Master 宕机情况下，消息无延迟，服务可用性与数据可用性都非常高
    > 缺点：性能比异步复制模式略低，大约低 10%左右，发送单个消息的 RT 会略高。

  以上 Broker 与 Slave 配对是通过指定相同的 brokerName 参数来配对，Master 的 BrokerId 必须是 0，Slave 的 BrokerId 必须是大于 0 的数。另外一个 Master 下面可以挂载多个 Slave，同一 Master 下的多个 Slave 通过指定不同的 BrokerId 来区分。

### 同步与异步刷盘的区别
- 写入PAGECACHE后立即返回(后台线程刷盘)
- 写入PAGECACHE后等待刷盘线程刷盘成功后返回

### 配置
2m-2s-async
```
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=0 # 0 master 1 slave
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER # SLAVE
flushDiskType=ASYNC_FLUSH

-------
brokerClusterName=DefaultCluster
brokerName=broker-b
brokerId=0 # 0 master 1 slave
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER # SLAVE
flushDiskType=ASYNC_FLUSH

```

### namesrv 配置
Wiki上是这么说的
>On default, the end point is:
http://jmenv.tbsite.net:8080/rocketmq/nsaddr
You may override jmenv.tbsite.net by this java option: rocketmq.namesrv.domain, You may also override nsaddr part by this java option: rocketmq.namesrv.domain.subgroup

然后我们看源码

```java
public class MixAll {
    public static final String ROCKETMQ_HOME_ENV = "ROCKETMQ_HOME";
    public static final String ROCKETMQ_HOME_PROPERTY = "rocketmq.home.dir";
    public static final String NAMESRV_ADDR_ENV = "NAMESRV_ADDR";
    public static final String NAMESRV_ADDR_PROPERTY = "rocketmq.namesrv.addr";
    public static final String MESSAGE_COMPRESS_LEVEL = "rocketmq.message.compressLevel";
    public static final String DEFAULT_NAMESRV_ADDR_LOOKUP = "jmenv.tbsite.net";
    public static final String WS_DOMAIN_NAME = System.getProperty("rocketmq.namesrv.domain", DEFAULT_NAMESRV_ADDR_LOOKUP);
    public static final String WS_DOMAIN_SUBGROUP = System.getProperty("rocketmq.namesrv.domain.subgroup", "nsaddr");
    // http://jmenv.tbsite.net:8080/rocketmq/nsaddr
    public static final String WS_ADDR = "http://" + WS_DOMAIN_NAME + ":8080/rocketmq/" + WS_DOMAIN_SUBGROUP;
}
```
只有WS_DOMAIN_NAME, WS_DOMAIN_SUBGROUP 是能配置的, 其他都是写死在代码. 如果生产上配置的话, 建议用http模式

#### 优先级
```
Programmatic Way > Java Options > Environment Variable > HTTP Endpoint
```

#### 可用性

由于消息分布在各个broker上，一旦某个broker宕机，则该broker上的消息读写都会受到影响。所以rocketmq提供了master/slave的结构，salve定时从master同步数据，如果master宕机，则slave提供消费服务，但是不能写入消息，此过程对应用透明，由rocketmq内部解决。

-  一旦某个broker master宕机，生产者和消费者多久才能发现？受限于rocketmq的网络连接机制，默认情况下，最多需要30秒，但这个时间可由应用设定参数来缩短时间。这个时间段内，发往该broker的消息都是失败的，而且该broker的消息无法消费，因为此时消费者不知道该broker已经挂掉。
- 消费者得到master宕机通知后，转向slave消费，但是slave不能保证master的消息100%都同步过来了，因此会有少量的消息丢失。但是消息最终不会丢的，一旦master恢复，未同步过去的消息会被消费掉。

#### 消息清理

- 扫描间隔
  默认10秒，由broker配置参数cleanResourceInterval决定
- 空间阈值
  物理文件不能无限制的一直存储在磁盘，当磁盘空间达到阈值时，不再接受消息，broker打印出日志，消息发送失败，阈值为固定值85%
- 清理时机
  默认每天凌晨4点，由broker配置参数deleteWhen决定；或者磁盘空间达到阈值
- 文件保留时长
  默认72小时，由broker配置参数fileReservedTime决定

#### 系统特性
- 大内存，内存越大性能越高，否则系统swap会成为性能瓶颈
- IO密集
- cpu load高，使用率低，因为cpu占用后，大部分时间在IO WAIT
- 磁盘可靠性要求高，为了兼顾安全和性能，采用RAID10阵列
- 磁盘读取速度要求快，要求高转速大容量磁盘


#### 对比阿里消息队列

| 项目    | 阿里消息队列    | 自建RocketMQ |
| :------------- | :-------------: | :---:|
| 监控      | 自带监控,附有短信提醒|没有主动监控通知功能, 需要自己搭建|
|运维成本|跟阿里消息队列故障率有关|采用2m-2s-async部署,Master 宕机，磁盘损坏情况，会丢失少量消息|
|面板一览|发布,订阅,消息,查询|发布,订阅,消息,查询(开源的RocketMQ-console-ng)|
|节点区域|华北,华南,华东,新加坡,公网|自建想去哪就去哪|
|安全性|对公网访问需要走阿里定制化的验证|无, 建议不开放公网访问|
|费用|高|按四台ECS2-4(322x4)|
|费用解释|包括请求, 上下行流量, 堆积等 [费用](https://www.aliyun.com/price/product#/mns/detail)|只有ECS费用|
