---
title: RocketMQ 顺序消息
date: 2019-11-21 23:10:00
tags: [java, RocketMQ]
---

# 什么是顺序消息 (RocketMQ 4.5.1)

## RocketMq支持的消息类型

1. Simple Message
2. **Order Message (FIFO)**
3. Broadcasting
4. Scheduled messages
5. Batch
6. Transaction messages

RocketMq 提供FIFO的有序消息. 局部消息有序, 可以保证同一个消息消费队列中消息被顺序消费,如果要保证全局顺序, 则只能将topic配置成一个queue了.

<!-- more -->

## 如何使用顺序消息

顺序消息有两点要注意的

1. 发送方, 也就是 producer 要保证发送到同一个queue
2. 消费方, 也就是 consumer 要保证是集群模式, 并且按顺序消费queue内容

看下官方例子

```java
public class OrderedProducer {
    public static void main(String[] args) throws Exception {
        //Instantiate with a producer group name.
        MQProducer producer = new DefaultMQProducer("example_group_name");
        //Launch the instance.
        producer.start();
        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 100; i++) {
            int orderId = i % 10;
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("TopicTestjjj", tags[i % tags.length], "KEY" + i,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
            @Override
            public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                // QueueSelector 保证发送到同一个queue
                Integer id = (Integer) arg;
                int index = id % mqs.size();
                return mqs.get(index);
            }
            }, orderId);

            System.out.printf("%s%n", sendResult);
        }
        //server shutdown
        producer.shutdown();
    }
}
```

消费者

```java
public class OrderedConsumer {
    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("example_group_name");

        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("TopicTest", "TagA || TagC || TagD");
        // 这里使用的 messageListener 是 MessageListenerOrderly
        // RocketMq 目前提供了两种, 另一个是 MessageListenerConcurrently
        consumer.registerMessageListener(new MessageListenerOrderly() {

            AtomicLong consumeTimes = new AtomicLong(0);
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                       ConsumeOrderlyContext context) {
                context.setAutoCommit(false);
                System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
                this.consumeTimes.incrementAndGet();
                if ((this.consumeTimes.get() % 2) == 0) {
                    // 标记成功, 在autoCommit为false的时候, 不会提交进度
                    return ConsumeOrderlyStatus.SUCCESS;
                } else if ((this.consumeTimes.get() % 3) == 0) {
                    // 标记回滚
                    return ConsumeOrderlyStatus.ROLLBACK;
                } else if ((this.consumeTimes.get() % 4) == 0) {
                    // 标记提交, 会提交消费进度
                    return ConsumeOrderlyStatus.COMMIT;
                } else if ((this.consumeTimes.get() % 5) == 0) {
                    // 标记等会重新消费
                    context.setSuspendCurrentQueueTimeMillis(3000);
                    return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                }
                return ConsumeOrderlyStatus.SUCCESS;

            }
        });

        consumer.start();

        System.out.printf("Consumer Started.%n");
    }
}
```

## RocketMQ 内部是怎么实现的

要先理解顺序消息是怎么实现的，先要大致了解 RocketMQ 本身消息的基本概念

### Topic

标识一类消息的逻辑名字，生产者传输消息到Topic，消费者从Topic拉去消息

### Message

消息就是被转递的内容。

### Message Queue

消息物理管理单位。一个Topic将有若干个Queue。
实际上，消息的发送与接收都是针对Message Queue的。默认的Producer消息发送，使用轮询的方式，选择Topic下的某一个Message Queue发送。Consumer消费的时候也会负载均衡地分配若干个Message Queue，只拉取对应Queue的消息。

### Message Model

消费模式，分两种

1. Clustering
    集群模式就是Consumer Group下的Consumer分摊消息，也就是一条消息只会被分配到组下的一个消费者
2. Broadcasting
    广播模式就是把消息投递每一个Consumer Group下的Consumer，因此组下每个消费者都会消费一次消息

{{<mermaid align="left" theme="dark" >}}
graph LR;

pg(producer group) --> MQ1[Topic1 Message Queue1]
pg(producer group) --> MQ2[Topic1 Message Queue2]
pg(producer group) --> MQ3[Topic1 Message Queue3]

MQ1 --> cg(consumer group)
MQ2 --> cg(consumer group)
MQ3 --> cg(consumer group)
{{</mermaid>}}

## 顺序保证

看上面RocketMQ结构上得知，要保证顺序消息，首先，producer要保证消息的顺序发送，那么可以把需要保证顺序的消息发送到同一个Queue中，消费者消费保证单线程消费同一个Queue即可。

RocketMQ就提供这种机制，producer 实现`MessageQueueSelector`来对消息进行分组顺序发送，消费者使用`MessageListenerOrderly`来保证消费的顺序性

### 具体实现

`MessageQueueSelector` 这个其实就是个Message Queue选择器策略的实现，自己控制消息发往哪个Message Queue

TL;DR

1. broker Message Queue 全局锁 (RebalanceImpl), 保证queue的分配唯一的消费者
    - 集群模式中 consumer 要先获取 broker 上 Message Queue 的锁，才能进行消费
    - 广播模式不可用
2. consumer 本地消息队列锁，持有锁操作此Queue
3. consumer 消费锁，持有锁才能消费消息
    保证消息顺序处理的细粒度锁

#### ConsumeMessageOrderlyService 主要方法

```java

public class ConsumeMessageOrderlyService implements ConsumeMessageService {

    public void start() {
        // 判断是否是集群模式, 只能在集群模式使用顺序消费
        if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())) {
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    ConsumeMessageOrderlyService.this.lockMQPeriodically();
                }
            }, 1000 * 1, ProcessQueue.REBALANCE_LOCK_INTERVAL, TimeUnit.MILLISECONDS);
        }
    }

    @Override
    public void submitConsumeRequest(
        final List<MessageExt> msgs,
        final ProcessQueue processQueue,
        final MessageQueue messageQueue,
        final boolean dispathToConsume) {
        if (dispathToConsume) {
            ConsumeRequest consumeRequest = new ConsumeRequest(processQueue, messageQueue);
            // 提交拉取的消息任务处理
            this.consumeExecutor.submit(consumeRequest);
        }
    }

    public synchronized boolean lockOneMQ(final MessageQueue mq) {
        if (!this.stopped) {
            return this.defaultMQPushConsumerImpl.getRebalanceImpl().lock(mq);
        }

        return false;
    }

    private void submitConsumeRequestLater(
        final ProcessQueue processQueue,
        final MessageQueue messageQueue,
        final long suspendTimeMillis
    ) {
        long timeMillis = suspendTimeMillis;
        if (timeMillis == -1) {
            timeMillis = this.defaultMQPushConsumer.getSuspendCurrentQueueTimeMillis();
        }

        if (timeMillis < 10) {
            timeMillis = 10;
        } else if (timeMillis > 30000) {
            timeMillis = 30000;
        }

        this.scheduledExecutorService.schedule(new Runnable() {

            @Override
            public void run() {
                ConsumeMessageOrderlyService.this.submitConsumeRequest(null, processQueue, messageQueue, true);
            }
        }, timeMillis, TimeUnit.MILLISECONDS);
    }

    public boolean processConsumeResult(
        final List<MessageExt> msgs,
        final ConsumeOrderlyStatus status,
        final ConsumeOrderlyContext context,
        final ConsumeRequest consumeRequest
    ) {
        boolean continueConsume = true;
        long commitOffset = -1L;
        // 如果开启了自动提交
        if (context.isAutoCommit()) {
            switch (status) {
                case COMMIT:
                case ROLLBACK:
                    // 如果开启了自动提交, COMMIT跟ROLLBACK都是不合法状态, 默认会SUCCESS
                    log.warn("the message queue consume result is illegal, we think you want to ack these message {}",
                        consumeRequest.getMessageQueue());
                case SUCCESS:
                    // 标记已经提交
                    commitOffset = consumeRequest.getProcessQueue().commit();
                    this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                    break;
                case SUSPEND_CURRENT_QUEUE_A_MOMENT:
                    this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                    if (checkReconsumeTimes(msgs)) {
                        // 设置重消费
                        consumeRequest.getProcessQueue().makeMessageToCosumeAgain(msgs);
                        this.submitConsumeRequestLater(
                            consumeRequest.getProcessQueue(),
                            consumeRequest.getMessageQueue(),
                            context.getSuspendCurrentQueueTimeMillis());
                        continueConsume = false;
                    } else {
                        // 重消费校验不过, 标记提交
                        commitOffset = consumeRequest.getProcessQueue().commit();
                    }
                    break;
                default:
                    break;
            }
        } else {
            // 没有开启自动提交
            switch (status) {
                case SUCCESS:
                    // success 不会提交确认已经提交
                    this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                    break;
                case COMMIT:
                    // commit 提交已经消费
                    commitOffset = consumeRequest.getProcessQueue().commit();
                    break;
                case ROLLBACK:
                    // 回滚消息
                    consumeRequest.getProcessQueue().rollback();
                    this.submitConsumeRequestLater(
                        consumeRequest.getProcessQueue(),
                        consumeRequest.getMessageQueue(),
                        context.getSuspendCurrentQueueTimeMillis());
                    continueConsume = false;
                    break;
                case SUSPEND_CURRENT_QUEUE_A_MOMENT:
                    this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                    if (checkReconsumeTimes(msgs)) {
                        // 重消费
                        consumeRequest.getProcessQueue().makeMessageToCosumeAgain(msgs);
                        this.submitConsumeRequestLater(
                            consumeRequest.getProcessQueue(),
                            consumeRequest.getMessageQueue(),
                            context.getSuspendCurrentQueueTimeMillis());
                        continueConsume = false;
                    }
                    break;
                default:
                    break;
            }
        }

        if (commitOffset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
            this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), commitOffset, false);
        }

        return continueConsume;
    }

    public ConsumerStatsManager getConsumerStatsManager() {
        return this.defaultMQPushConsumerImpl.getConsumerStatsManager();
    }

    private int getMaxReconsumeTimes() {
        // default reconsume times: Integer.MAX_VALUE
        if (this.defaultMQPushConsumer.getMaxReconsumeTimes() == -1) {
            return Integer.MAX_VALUE;
        } else {
            return this.defaultMQPushConsumer.getMaxReconsumeTimes();
        }
    }

    private boolean checkReconsumeTimes(List<MessageExt> msgs) {
        boolean suspend = false;
        if (msgs != null && !msgs.isEmpty()) {
            for (MessageExt msg : msgs) {
                if (msg.getReconsumeTimes() >= getMaxReconsumeTimes()) {
                    MessageAccessor.setReconsumeTime(msg, String.valueOf(msg.getReconsumeTimes()));
                    if (!sendMessageBack(msg)) {
                        suspend = true;
                        msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
                    }
                } else {
                    suspend = true;
                    msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
                }
            }
        }
        return suspend;
    }

    public boolean sendMessageBack(final MessageExt msg) {
        try {
            // max reconsume times exceeded then send to dead letter queue.
            Message newMsg = new Message(MixAll.getRetryTopic(this.defaultMQPushConsumer.getConsumerGroup()), msg.getBody());
            String originMsgId = MessageAccessor.getOriginMessageId(msg);
            MessageAccessor.setOriginMessageId(newMsg, UtilAll.isBlank(originMsgId) ? msg.getMsgId() : originMsgId);
            newMsg.setFlag(msg.getFlag());
            MessageAccessor.setProperties(newMsg, msg.getProperties());
            MessageAccessor.putProperty(newMsg, MessageConst.PROPERTY_RETRY_TOPIC, msg.getTopic());
            MessageAccessor.setReconsumeTime(newMsg, String.valueOf(msg.getReconsumeTimes()));
            MessageAccessor.setMaxReconsumeTimes(newMsg, String.valueOf(getMaxReconsumeTimes()));
            newMsg.setDelayTimeLevel(3 + msg.getReconsumeTimes());

            this.defaultMQPushConsumer.getDefaultMQPushConsumerImpl().getmQClientFactory().getDefaultMQProducer().send(newMsg);
            return true;
        } catch (Exception e) {
            log.error("sendMessageBack exception, group: " + this.consumerGroup + " msg: " + msg.toString(), e);
        }

        return false;
    }

    public void resetNamespace(final List<MessageExt> msgs) {
        for (MessageExt msg : msgs) {
            if (StringUtils.isNotEmpty(this.defaultMQPushConsumer.getNamespace())) {
                msg.setTopic(NamespaceUtil.withoutNamespace(msg.getTopic(), this.defaultMQPushConsumer.getNamespace()));
            }
        }
    }

    class ConsumeRequest implements Runnable {
        private final ProcessQueue processQueue;
        private final MessageQueue messageQueue;

        public ConsumeRequest(ProcessQueue processQueue, MessageQueue messageQueue) {
            this.processQueue = processQueue;
            this.messageQueue = messageQueue;
        }

        public ProcessQueue getProcessQueue() {
            return processQueue;
        }

        public MessageQueue getMessageQueue() {
            return messageQueue;
        }

        // 处理进来的消息
        @Override
        public void run() {
            if (this.processQueue.isDropped()) {
                log.warn("run, the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                return;
            }

            final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
            // 本地消息队列锁, 控制在单队列消费中,只占用一个线程
            synchronized (objLock) {
                if (MessageModel.BROADCASTING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                    || (this.processQueue.isLocked() && !this.processQueue.isLockExpired())) {
                    // 广播类型 或者 队列锁了,并且锁有效
                    final long beginTime = System.currentTimeMillis();
                    // 循环判断 continueConsume
                    for (boolean continueConsume = true; continueConsume; ) {
                        if (this.processQueue.isDropped()) {
                            log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                            break;
                        }

                        if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                            && !this.processQueue.isLocked()) {
                            // 集群模式 并 没锁, 重消费
                            log.warn("the message queue not locked, so consume later, {}", this.messageQueue);
                            ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
                            break;
                        }

                        if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                            && this.processQueue.isLockExpired()) {
                            // 集群模式 并 锁失效, 重消费
                            log.warn("the message queue lock expired, so consume later, {}", this.messageQueue);
                            ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
                            break;
                        }

                        long interval = System.currentTimeMillis() - beginTime;
                        if (interval > MAX_TIME_CONSUME_CONTINUOUSLY) {
                            ConsumeMessageOrderlyService.this.submitConsumeRequestLater(processQueue, messageQueue, 10);
                            break;
                        }

                        final int consumeBatchSize =
                            ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();

                        // 拉取待处理消息
                        List<MessageExt> msgs = this.processQueue.takeMessags(consumeBatchSize);
                        defaultMQPushConsumerImpl.resetRetryAndNamespace(msgs, defaultMQPushConsumer.getConsumerGroup());
                        if (!msgs.isEmpty()) {
                            final ConsumeOrderlyContext context = new ConsumeOrderlyContext(this.messageQueue);

                            ConsumeOrderlyStatus status = null;

                            ConsumeMessageContext consumeMessageContext = null;
                            if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                                consumeMessageContext = new ConsumeMessageContext();
                                consumeMessageContext
                                    .setConsumerGroup(ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumerGroup());
                                consumeMessageContext.setNamespace(defaultMQPushConsumer.getNamespace());
                                consumeMessageContext.setMq(messageQueue);
                                consumeMessageContext.setMsgList(msgs);
                                consumeMessageContext.setSuccess(false);
                                // init the consume context type
                                consumeMessageContext.setProps(new HashMap<String, String>());
                                ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
                            }

                            long beginTimestamp = System.currentTimeMillis();
                            ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
                            boolean hasException = false;
                            try {
                                this.processQueue.getLockConsume().lock();
                                if (this.processQueue.isDropped()) {
                                    log.warn("consumeMessage, the message queue not be able to consume, because it's dropped. {}",
                                        this.messageQueue);
                                    break;
                                }
                                // 调用处理的方法, 返回处理状态
                                status = messageListener.consumeMessage(Collections.unmodifiableList(msgs), context);
                            } catch (Throwable e) {
                                log.warn("consumeMessage exception: {} Group: {} Msgs: {} MQ: {}",
                                    RemotingHelper.exceptionSimpleDesc(e),
                                    ConsumeMessageOrderlyService.this.consumerGroup,
                                    msgs,
                                    messageQueue);
                                hasException = true;
                            } finally {
                                this.processQueue.getLockConsume().unlock();
                            }

                            if (null == status
                                || ConsumeOrderlyStatus.ROLLBACK == status
                                || ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
                                // 返回状态有问题, 记录日志
                                log.warn("consumeMessage Orderly return not OK, Group: {} Msgs: {} MQ: {}",
                                    ConsumeMessageOrderlyService.this.consumerGroup,
                                    msgs,
                                    messageQueue);
                            }

                            long consumeRT = System.currentTimeMillis() - beginTimestamp;
                            if (null == status) {
                                if (hasException) {
                                    returnType = ConsumeReturnType.EXCEPTION;
                                } else {
                                    returnType = ConsumeReturnType.RETURNNULL;
                                }
                            } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
                                returnType = ConsumeReturnType.TIME_OUT;
                            } else if (ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
                                returnType = ConsumeReturnType.FAILED;
                            } else if (ConsumeOrderlyStatus.SUCCESS == status) {
                                returnType = ConsumeReturnType.SUCCESS;
                            }

                            if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                                consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
                            }

                            if (null == status) {
                                status = ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                            }

                            if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                                consumeMessageContext.setStatus(status.toString());
                                consumeMessageContext
                                    .setSuccess(ConsumeOrderlyStatus.SUCCESS == status || ConsumeOrderlyStatus.COMMIT == status);
                                ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
                            }

                            ConsumeMessageOrderlyService.this.getConsumerStatsManager()
                                .incConsumeRT(ConsumeMessageOrderlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);
                            // 处理返回的状态结果, 判断是否继续消费
                            continueConsume = ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this);
                        } else {
                            continueConsume = false;
                        }
                    }
                } else {
                    if (this.processQueue.isDropped()) {
                        log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                        return;
                    }

                    ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 100);
                }
            }
        }

    }

}

```

### 消费结果处理

在没有开启自动提交的情况下

1. SUCCESS：消费成功但不提交。
2. ROLLBACK：消费失败，消费回滚。
3. COMMIT：消费成功提交并且提交。
4. SUSPEND_CURRENT_QUEUE_A_MOMENT：消费失败，挂起消费队列一会会，稍后继续消费。

开启的话

1. SUCCESS：消费成功并提交
2. SUSPEND_CURRENT_QUEUE_A_MOMENT：消费失败，挂起消费队列一会会，稍后继续消费。

超过重试次数，RocketMQ会把消息发到死信队列里面去(broker 判断最大重试次数，原本应该是放入重试队列，超过就扔到死信队列了)

## 总结

1. 发送方保证Message Queue里面的消息是顺序的
2. 消费方使用ConsumeOrderlyStatus消息消息
3. 处理消费结果状态，保证消息的状态正确

## 问题

1. 上面描述的是局部顺序保证，要全局顺序保证只能Topic下只有一个Message Queue。
