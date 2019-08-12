---
title: RocketMQ 事务消息
date: 2019-08-12 22:10:00
tags: [java, RocketMQ]
---

# 什么是事务消息 (RocketMQ 4.5.1)

RocketMQ 中, 可以把事务消息当成两阶段提交的消息, 用来确保分布式系统的最终一致性.事务消息确保可以以原子方式执行本地事务和消息发送.

<!-- more -->

## 如何使用事务消息

``` java

public class TransactionProducer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        TransactionListener transactionListener = new TransactionListenerImpl();
        TransactionMQProducer producer = new TransactionMQProducer("please_rename_unique_group_name");
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("client-transaction-msg-check-thread");
                return thread;
            }
        });

        producer.setExecutorService(executorService);
        producer.setTransactionListener(transactionListener);
        producer.start();

        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 10; i++) {
            try {
                Message msg =
                    new Message("TopicTest1234", tags[i % tags.length], "KEY" + i,
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.sendMessageInTransaction(msg, null);
                System.out.printf("%s%n", sendResult);

                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }

        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }
        producer.shutdown();
    }
}
```

从官方例子可以看出, 主要是 `TransactionMQProducer` 作为 producer 的实现类, `TransactionListener` 做为本地事务处理的接口

```java
public interface TransactionListener {
    /**
     * When send transactional prepare(half) message succeed, this method will be invoked to execute local transaction.
     *
     * @param msg Half(prepare) message
     * @param arg Custom business parameter
     * @return Transaction state
     */
    LocalTransactionState executeLocalTransaction(final Message msg, final Object arg);

    /**
     * When no response to prepare(half) message. broker will send check message to check the transaction status, and this
     * method will be invoked to get local transaction status.
     *
     * @param msg Check message
     * @return Transaction state
     */
    LocalTransactionState checkLocalTransaction(final MessageExt msg);
}
```

## RocketMQ 内部是怎么实现的

```java
    // 发送事务消息
    public TransactionSendResult sendMessageInTransaction(final Message msg,
                                                          final LocalTransactionExecuter localTransactionExecuter, final Object arg)
        throws MQClientException {
        TransactionListener transactionListener = getCheckListener();
        if (null == localTransactionExecuter && null == transactionListener) {
            throw new MQClientException("tranExecutor is null", null);
        }
        Validators.checkMessage(msg, this.defaultMQProducer);

        SendResult sendResult = null;
        // 设置消息为事务消息
        // 设置 property TRAN_MSG:true
        MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");
        MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());
        try {
            sendResult = this.send(msg);
        } catch (Exception e) {
            throw new MQClientException("send message Exception", e);
        }

        LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
        Throwable localException = null;
        switch (sendResult.getSendStatus()) {
            case SEND_OK: {
                try {
                    if (sendResult.getTransactionId() != null) {
                        msg.putUserProperty("__transactionId__", sendResult.getTransactionId());
                    }
                    String transactionId = msg.getProperty(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
                    if (null != transactionId && !"".equals(transactionId)) {
                        msg.setTransactionId(transactionId);
                    }
                    // 处理本地事务
                    if (null != localTransactionExecuter) {
                        localTransactionState = localTransactionExecuter.executeLocalTransactionBranch(msg, arg);
                    } else if (transactionListener != null) {
                        log.debug("Used new transaction API");
                        localTransactionState = transactionListener.executeLocalTransaction(msg, arg);
                    }
                    if (null == localTransactionState) {
                        localTransactionState = LocalTransactionState.UNKNOW;
                    }

                    if (localTransactionState != LocalTransactionState.COMMIT_MESSAGE) {
                        log.info("executeLocalTransactionBranch return {}", localTransactionState);
                        log.info(msg.toString());
                    }
                } catch (Throwable e) {
                    log.info("executeLocalTransactionBranch exception", e);
                    log.info(msg.toString());
                    localException = e;
                }
            }
            break;
            case FLUSH_DISK_TIMEOUT:
            case FLUSH_SLAVE_TIMEOUT:
            case SLAVE_NOT_AVAILABLE:
                localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;
                break;
            default:
                break;
        }

        try {
            // 本地消息处理后
            this.endTransaction(sendResult, localTransactionState, localException);
        } catch (Exception e) {
            log.warn("local transaction execute " + localTransactionState + ", but end broker transaction failed", e);
        }

        TransactionSendResult transactionSendResult = new TransactionSendResult();
        transactionSendResult.setSendStatus(sendResult.getSendStatus());
        transactionSendResult.setMessageQueue(sendResult.getMessageQueue());
        transactionSendResult.setMsgId(sendResult.getMsgId());
        transactionSendResult.setQueueOffset(sendResult.getQueueOffset());
        transactionSendResult.setTransactionId(sendResult.getTransactionId());
        transactionSendResult.setLocalTransactionState(localTransactionState);
        return transactionSendResult;
    }
```

client 主要的工作

1. 为消息添加事务消息标识属性 TRAN_MSG:True
2. 添加属性 PGROUP 生产者所在的消息组
3. 调用默认的生产者实现`DefaultMQProducerImpl`, 判断如果是事务消息, 则标记消息类型为`Trans_Msg_Half` 预提交阶段(半消息)
4. 发送成功后, 执行本地事务
5. 根据本地事务返回的结果执行`endTransaction`
6. `endTransaction` 里面实际是把本地事务执行结果, 设置在 CommitOrRollback 头中, 再发送给broker, 标识为 `END_TRANSACTION`.

endTransaction

```java
    public void endTransaction(
        final SendResult sendResult,
        final LocalTransactionState localTransactionState,
        final Throwable localException) throws RemotingException, MQBrokerException, InterruptedException, UnknownHostException {
        final MessageId id;
        if (sendResult.getOffsetMsgId() != null) {
            id = MessageDecoder.decodeMessageId(sendResult.getOffsetMsgId());
        } else {
            id = MessageDecoder.decodeMessageId(sendResult.getMsgId());
        }
        String transactionId = sendResult.getTransactionId();
        final String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(sendResult.getMessageQueue().getBrokerName());
        EndTransactionRequestHeader requestHeader = new EndTransactionRequestHeader();
        requestHeader.setTransactionId(transactionId);
        requestHeader.setCommitLogOffset(id.getOffset());
        switch (localTransactionState) {
            case COMMIT_MESSAGE: // 本地事务处理成功, 设置 commit ,提交事务
                requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
                break;
            case ROLLBACK_MESSAGE: // 本地事务处理失败, 设置 rollback, 回滚事务
                requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
                break;
            case UNKNOW:
                requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
                break;
            default:
                break;
        }

        requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
        requestHeader.setTranStateTableOffset(sendResult.getQueueOffset());
        requestHeader.setMsgId(sendResult.getMsgId());
        String remark = localException != null ? ("executeLocalTransactionBranch exception: " + localException.toString()) : null;
        // end transaction broker EndTransactionProcessor
        this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, requestHeader, remark,
            this.defaultMQProducer.getSendMsgTimeout());
    }
```

总流程

``` sh
                                  +------------------------+         +
                                  |                        |         | 1. add property TRAN_MSG:TRUE
sendMessageInTransation +-------->+  TransationMQProducer  +<--------+ 2. add property PGROUP
                                  |                        |         |
                                  +-----------+------------+         +
                                              |
                                              |
                              +---------------v----------------+     +
                              |                                |     | 1. setMsgType Trans_Msg_Half
                              |     DefaultMQProducerImpl#Send +<----+
                              |                                |     |
                              +---------------+----------------+     +
                                              |
                                              | Sync
                                              |
                              +---------------v----------------+     +
                              |                                |     | 1. broker
                              |    SendMessageProcessor        +<----+
                              |                                |     |
                              +---------------+----------------+     +
                                              |
                                              | send Success
                                              |
                                  +-----------v------------+         +
                                  |                        |         | 1. new/old api
                                  | exec local transaction +<--------+
                                  |                        |         |
                                  +-----------+------------+         +
                                              |
                                              | return local state
                                              |
                                    +---------v------------+
                                    |                      |
                                    |  1. COMMIT_MESSAGE   |
                                    |  2. ROLLBACK_MESSAGE |
                                    |  3. UNKNOW           |
                                    |                      |
                                    +---------+------------+
                                              |
                                              | exec
                                              |
                                      +-------v----------+           +
                                      |                  |           | 1. set EndTransactionRequestHeader
                                      |  endTransaction  +<----------+ 2. endTransactionOneWay
                                      |                  |           | 3. cmd END_TRANSACTION
                                      +------------------+           +

```

## broker 的工作

1. 判断是否是事务消息
2. 处理半消息
    2.1 备份原消息的主题跟原队列id
    2.2 重新设置消息topic为`RMQ_SYS_TRANS_HALF_TOPIC`
3. 持久化

    ```java
    // 事务消息处理
    private MessageExtBrokerInner parseHalfMessageInner(MessageExtBrokerInner msgInner) {
        // 把原消息的 topic 备份到 property
        MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_TOPIC, msgInner.getTopic());
        // 把原消息的 queue 备份到 property
        MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_QUEUE_ID,
            String.valueOf(msgInner.getQueueId()));
        msgInner.setSysFlag(
            MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), MessageSysFlag.TRANSACTION_NOT_TYPE));
        // 设置特殊的半消息 topic RMQ_SYS_TRANS_HALF_TOPIC
        msgInner.setTopic(TransactionalMessageUtil.buildHalfTopic());
        // 设置特殊的 queueId 固定为 0
        msgInner.setQueueId(0);
        msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
        return msgInner;
    }
    ```

4. EndTransactionRequest, 处理 commit 和 rollback 消息 (client#endTransaction)
    4.1 如果是 commit 消息, **还原原有主题跟队列**
    4.2 跟普通消息一样.
    4.3 存储已经处理的消息在`RMQ_SYS_TRANS_OP_HALF_TOPIC`
    4.4 如果是 rollback 消息, 讲消息写入 `RMQ_SYS_TRANS_OP_HALF_TOPIC`, 标记删除tag

    ```java
    OperationResult result = new OperationResult();
        if (MessageSysFlag.TRANSACTION_COMMIT_TYPE == requestHeader.getCommitOrRollback()) {
            // 如果是commit
            // 这里实际是通过 offSet 查到commitLog requestHeader.getCommitLogOffset()
            result = this.brokerController.getTransactionalMessageService().commitMessage(requestHeader);
            if (result.getResponseCode() == ResponseCode.SUCCESS) {
                RemotingCommand res = checkPrepareMessage(result.getPrepareMessage(), requestHeader);
                if (res.getCode() == ResponseCode.SUCCESS) {
                    // 消息还原
                    MessageExtBrokerInner msgInner = endMessageTransaction(result.getPrepareMessage());
                    msgInner.setSysFlag(MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), requestHeader.getCommitOrRollback()));
                    msgInner.setQueueOffset(requestHeader.getTranStateTableOffset());
                    msgInner.setPreparedTransactionOffset(requestHeader.getCommitLogOffset());
                    msgInner.setStoreTimestamp(result.getPrepareMessage().getStoreTimestamp());
                    // 最终消息处理还是走原有的普通消息 this.brokerController.getMessageStore().putMessage
                    RemotingCommand sendResult = sendFinalMessage(msgInner);
                    if (sendResult.getCode() == ResponseCode.SUCCESS) {
                        // 删除半消息 实际是存储在 RMQ_SYS_TRANS_OP_HALF_TOPIC 中, 表示事务消息已经被处理了
                        this.brokerController.getTransactionalMessageService().deletePrepareMessage(result.getPrepareMessage());
                    }
                    return sendResult;
                }
                return res;
            }
        } else if (MessageSysFlag.TRANSACTION_ROLLBACK_TYPE == requestHeader.getCommitOrRollback()) {
            // 如果是回滚
            result = this.brokerController.getTransactionalMessageService().rollbackMessage(requestHeader);
            if (result.getResponseCode() == ResponseCode.SUCCESS) {
                RemotingCommand res = checkPrepareMessage(result.getPrepareMessage(), requestHeader);
                if (res.getCode() == ResponseCode.SUCCESS) {
                    this.brokerController.getTransactionalMessageService().deletePrepareMessage(result.getPrepareMessage());
                }
                return res;
            }
        }
    ```

## 消息回查

1. 半消息提交到topic`RMQ_SYS_TRANS_HALF_TOPIC`
2. 消息处理后(commit/rollback)提交到topic`RMQ_SYS_TRANS_OP_HALF_TOPIC`

```java
    // TransactionalMessageCheckService.java
    @Override
    public void run() {
        log.info("Start transaction check service thread!");
        // 默认 60s
        long checkInterval = brokerController.getBrokerConfig().getTransactionCheckInterval();
        while (!this.isStopped()) {
            this.waitForRunning(checkInterval);
        }
        log.info("End transaction check service thread!");
    }

    @Override
    protected void onWaitEnd() {
        long timeout = brokerController.getBrokerConfig().getTransactionTimeOut();
        // 事务最大检测次数
        int checkMax = brokerController.getBrokerConfig().getTransactionCheckMax();
        long begin = System.currentTimeMillis();
        log.info("Begin to check prepare message, begin time:{}", begin);
        this.brokerController.getTransactionalMessageService().check(timeout, checkMax, this.brokerController.getTransactionalMessageCheckListener());
        log.info("End to check prepare message, consumed time:{}", System.currentTimeMillis() - begin);
    }
```

1. RocketMQ 在 broker 初始化的时候, 通过 `TransactionalMessageCheckService` 线程去定时检测 `RMQ_SYS_TRANS_HALF_TOPIC` 的消息, 回查消息事务状态
2. 遍历 Half `RMQ_SYS_TRANS_HALF_TOPIC` 队列消息
3. 根据 OP `RMQ_SYS_TRANS_OP_HALF_TOPIC` 队列判断是否已经处理
4. 更新 Half, OP 队列进度
5. 根据 OP 队列当前更新进度, 往后获取32条消息
6. 判断是否需要发送回查消息 (本地事务超时时间,消息有效性,消息是否发送回查消息)
7. 新线程发送回查
8. 更新 Half, OP 队列处理进度

``` sh
                                                                                              +
                                                                                              | 1. RMQ_SYS_TRANS_HALF_TOPIC
                                               +--------------------------------------+       | 2. RMQ_SYS_TRANS_OP_HALF_TOPIC
                                               |                                      |       | 3. read more OP
TransactionalMessageCheckService +-----------> | TransactionMessageCheckService#check | <-----+ 4. isNeedCheck
                                               |                                      |       | 5. check async
                                               +------------------+-------------------+       | 6. update offset
                                                                  |                           +
                                                                  |
                                                                  |
                                                                  v
                                       +--------------------------+-------------------------------+      +
                                       |                                                          |      |
                                       | AbstractTransactionalMessageCheckListener#resolveHalfMsg | <----+ 1. sendCheckMessage
                                       |                                                          |      |
                                       +---------------------------+------------------------------+      +
                                                                   |
                                                                   |
                                                                   |
                                                                   |
                                                   +---------------v---------------+
                                                   |                               |
                                                   | checkProducerTransactionState |
                                                   |                               |
                                                   +---------------+---------------+
                                                                   |
                                                                   |   client site
                                                                   |
                                            +----------------------v-----------------------+
                                            |                                              |
                                            | ClientRemotingProcessor#checkTransationState |
                                            |                                              |
                                            +-----------------------+----------------------+
                                                                    |
                                                                    |
                                                                    |
                                            +-----------------------v-----------------------+
                                            |                                               |
                                            |  DefaultMQProducerImpl#checkTransactionState  |
                                            |                                               |
                                            +----------------------+------------------------+
                                                                   |
                                                                   |
                                                                   |
                                             +---------------------v---------------------+
                                             |                                           |
                                             | TransactionListener#checkLocalTransaction |
                                             |                                           |
                                             +--------------------+----------------------+
                                                                  |
                                                                  |     send back to broker
                                                                  |
                                                        +---------v---------+
                                                        |                   |
                                                        |  END_TRANSACTION  |
                                                        |                   |
                                                        +-------------------+

```

1. client 接受到回查消息
2. 开启新线程去回查
3. 主要是 `transactionListener#checkLocalTransaction` 本地事务检查
4. 发送 COMMIT_MESSAGE、ROLLBACK_MESSAGE、UNKNOW中的一个，然后向Broker发送END_TRANSACTION命令即可

```java
// TransactionalMessageCheckService 定时检查方法
    @Override
    public void check(long transactionTimeout, int transactionCheckMax,
        AbstractTransactionalMessageCheckListener listener) {
        try {
            String topic = MixAll.RMQ_SYS_TRANS_HALF_TOPIC;
            Set<MessageQueue> msgQueues = transactionalMessageBridge.fetchMessageQueues(topic);
            if (msgQueues == null || msgQueues.size() == 0) {
                log.warn("The queue of topic is empty :" + topic);
                return;
            }
            log.debug("Check topic={}, queues={}", topic, msgQueues);
            // 遍历 Half 消息
            for (MessageQueue messageQueue : msgQueues) {
                long startTime = System.currentTimeMillis();
                MessageQueue opQueue = getOpQueue(messageQueue);
                long halfOffset = transactionalMessageBridge.fetchConsumeOffset(messageQueue);
                long opOffset = transactionalMessageBridge.fetchConsumeOffset(opQueue);
                log.info("Before check, the queue={} msgOffset={} opOffset={}", messageQueue, halfOffset, opOffset);
                if (halfOffset < 0 || opOffset < 0) {
                    log.error("MessageQueue: {} illegal offset read: {}, op offset: {},skip this queue", messageQueue,
                        halfOffset, opOffset);
                    continue;
                }

                List<Long> doneOpOffset = new ArrayList<>();
                HashMap<Long, Long> removeMap = new HashMap<>();
                // 从RMQ_SYS_TRANS_OP_HALF_TOPIC主题中拉取32条，
                // 如果拉取的消息队列偏移量大于等于RMQ_SYS_TRANS_HALF_TOPIC#queueId当前的处理进度时，
                // 会添加到removeMap中，表示已处理过。
                PullResult pullResult = fillOpRemoveMap(removeMap, opQueue, opOffset, halfOffset, doneOpOffset);
                if (null == pullResult) {
                    log.error("The queue={} check msgOffset={} with opOffset={} failed, pullResult is null",
                        messageQueue, halfOffset, opOffset);
                    continue;
                }
                // single thread
                // 获取空消息的次数
                int getMessageNullCount = 1;
                // 当前处理的最新偏移量
                long newOffset = halfOffset;
                // 当前处理消息偏移量
                long i = halfOffset;
                while (true) {
                    // 限制每次最多处理的时间
                    if (System.currentTimeMillis() - startTime > MAX_PROCESS_TIME_LIMIT) {
                        log.info("Queue={} process time reach max={}", messageQueue, MAX_PROCESS_TIME_LIMIT);
                        break;
                    }
                    // 如果removeMap中包含当前处理的消息，则继续下一条
                    if (removeMap.containsKey(i)) {
                        log.info("Half offset {} has been committed/rolled back", i);
                        removeMap.remove(i);
                    } else {
                        GetResult getResult = getHalfMsg(messageQueue, i);
                        MessageExt msgExt = getResult.getMsg();
                        if (msgExt == null) {
                            // 默认重试次数
                            if (getMessageNullCount++ > MAX_RETRY_COUNT_WHEN_HALF_NULL) {
                                break;
                            }
                            // 没新消息
                            if (getResult.getPullResult().getPullStatus() == PullStatus.NO_NEW_MSG) {
                                log.debug("No new msg, the miss offset={} in={}, continue check={}, pull result={}", i,
                                    messageQueue, getMessageNullCount, getResult.getPullResult());
                                break;
                            } else {
                                log.info("Illegal offset, the miss offset={} in={}, continue check={}, pull result={}",
                                    i, messageQueue, getMessageNullCount, getResult.getPullResult());
                                i = getResult.getPullResult().getNextBeginOffset();
                                newOffset = i;
                                // 重置偏移量, 重新拉取
                                continue;
                            }
                        }
                        // 判断是否丢弃, 跳过
                        // needDiscard 最大回查次数 TRANSACTION_CHECK_TIMES
                        // needSkip 事务超过文件过期时间 72h
                        if (needDiscard(msgExt, transactionCheckMax) || needSkip(msgExt)) {
                            listener.resolveDiscardMsg(msgExt);
                            newOffset = i + 1;
                            i++;
                            continue;
                        }
                        if (msgExt.getStoreTimestamp() >= startTime) {
                            log.debug("Fresh stored. the miss offset={}, check it later, store={}", i,
                                new Date(msgExt.getStoreTimestamp()));
                            break;
                        }
                        // 消息已经保存时间
                        long valueOfCurrentMinusBorn = System.currentTimeMillis() - msgExt.getBornTimestamp();
                        // 立刻检查事务的时间
                        long checkImmunityTime = transactionTimeout;
                        // transactionTimeout 事务消息的超时时间 (最短可回查时间)
                        // 该时间就是假设事务消息发送成功后，应用程序事务提交的时间，
                        // 在这段时间内，RocketMQ任务事务未提交，故不应该在这个时间段向应用程序发送回查请求。
                        String checkImmunityTimeStr = msgExt.getUserProperty(MessageConst.PROPERTY_CHECK_IMMUNITY_TIME_IN_SECONDS);
                        if (null != checkImmunityTimeStr) {
                            // 事务消息制定了事务消息过期时间
                            checkImmunityTime = getImmunityTime(checkImmunityTimeStr, transactionTimeout);
                            if (valueOfCurrentMinusBorn < checkImmunityTime) {
                                if (checkPrepareQueueOffset(removeMap, doneOpOffset, msgExt)) {
                                    newOffset = i + 1;
                                    i++;
                                    continue;
                                }
                            }
                        } else {
                            if ((0 <= valueOfCurrentMinusBorn) && (valueOfCurrentMinusBorn < checkImmunityTime)) {
                                log.debug("New arrived, the miss offset={}, check it later checkImmunity={}, born={}", i,
                                    checkImmunityTime, new Date(msgExt.getBornTimestamp()));
                                break;
                            }
                        }
                        List<MessageExt> opMsg = pullResult.getMsgFoundList();
                        boolean isNeedCheck = (opMsg == null && valueOfCurrentMinusBorn > checkImmunityTime)
                            || (opMsg != null && (opMsg.get(opMsg.size() - 1).getBornTimestamp() - startTime > transactionTimeout))
                            || (valueOfCurrentMinusBorn <= -1);

                        if (isNeedCheck) {
                            // 如果需要发送事务状态回查消息，则先将消息再次发送到RMQ_SYS_TRANS_HALF_TOPIC主题中，
                            // 发送成功则返回true，否则返回false
                            // 成功会将该消息的queueOffset、commitLogOffset设置为重新存入的偏移量
                            if (!putBackHalfMsgQueue(msgExt, i)) {
                                continue;
                            }
                            // 需要回查半消息
                            // 回查消息是用另个一线程发送的
                            listener.resolveHalfMsg(msgExt);
                        } else {
                            pullResult = fillOpRemoveMap(removeMap, opQueue, pullResult.getNextBeginOffset(), halfOffset, doneOpOffset);
                            log.info("The miss offset:{} in messageQueue:{} need to get more opMsg, result is:{}", i,
                                messageQueue, pullResult);
                            continue;
                        }
                    }
                    newOffset = i + 1;
                    i++;
                }
                if (newOffset != halfOffset) {
                    // 保存(Prepare)消息队列的回查进度。
                    transactionalMessageBridge.updateConsumeOffset(messageQueue, newOffset);
                }
                long newOpOffset = calculateOpOffset(doneOpOffset, opOffset);
                if (newOpOffset != opOffset) {
                    // 保存处理队列（op）的进度。
                    transactionalMessageBridge.updateConsumeOffset(opQueue, newOpOffset);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
            log.error("Check error", e);
        }

    }
```

client 回查事务消息

```java
    // 事务消息回查
    @Override
    public void checkTransactionState(final String addr, final MessageExt msg,
        final CheckTransactionStateRequestHeader header) {
        // 开新线程检查
        Runnable request = new Runnable() {
            private final String brokerAddr = addr;
            private final MessageExt message = msg;
            private final CheckTransactionStateRequestHeader checkRequestHeader = header;
            private final String group = DefaultMQProducerImpl.this.defaultMQProducer.getProducerGroup();

            @Override
            public void run() {
                TransactionCheckListener transactionCheckListener = DefaultMQProducerImpl.this.checkListener();
                TransactionListener transactionListener = getCheckListener();
                if (transactionCheckListener != null || transactionListener != null) {
                    LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
                    Throwable exception = null;
                    try {
                        if (transactionCheckListener != null) {
                            // 本地事务检查
                            localTransactionState = transactionCheckListener.checkLocalTransactionState(message);
                        } else if (transactionListener != null) {
                            log.debug("Used new check API in transaction message");
                            // 本地事务检查
                            localTransactionState = transactionListener.checkLocalTransaction(message);
                        } else {
                            log.warn("CheckTransactionState, pick transactionListener by group[{}] failed", group);
                        }
                    } catch (Throwable e) {
                        log.error("Broker call checkTransactionState, but checkLocalTransactionState exception", e);
                        exception = e;
                    }
                    // 发送 COMMIT_MESSAGE、ROLLBACK_MESSAGE、UNKNOW中的一个，然后向Broker发送END_TRANSACTION命令即可
                    this.processTransactionState(
                        localTransactionState,
                        group,
                        exception);
                } else {
                    log.warn("CheckTransactionState, pick transactionCheckListener by group[{}] failed", group);
                }
            }

            private void processTransactionState(
                final LocalTransactionState localTransactionState,
                final String producerGroup,
                final Throwable exception) {
                final EndTransactionRequestHeader thisHeader = new EndTransactionRequestHeader();
                thisHeader.setCommitLogOffset(checkRequestHeader.getCommitLogOffset());
                thisHeader.setProducerGroup(producerGroup);
                thisHeader.setTranStateTableOffset(checkRequestHeader.getTranStateTableOffset());
                thisHeader.setFromTransactionCheck(true);

                String uniqueKey = message.getProperties().get(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
                if (uniqueKey == null) {
                    uniqueKey = message.getMsgId();
                }
                thisHeader.setMsgId(uniqueKey);
                thisHeader.setTransactionId(checkRequestHeader.getTransactionId());
                switch (localTransactionState) {
                    case COMMIT_MESSAGE:
                        thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
                        break;
                    case ROLLBACK_MESSAGE:
                        thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
                        log.warn("when broker check, client rollback this transaction, {}", thisHeader);
                        break;
                    case UNKNOW:
                        thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
                        log.warn("when broker check, client does not know this transaction state, {}", thisHeader);
                        break;
                    default:
                        break;
                }

                String remark = null;
                if (exception != null) {
                    remark = "checkLocalTransactionState Exception: " + RemotingHelper.exceptionSimpleDesc(exception);
                }

                try {
                    DefaultMQProducerImpl.this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr, thisHeader, remark,
                        3000);
                } catch (Exception e) {
                    log.error("endTransactionOneway exception", e);
                }
            }
        };

        this.checkExecutor.submit(request);
    }
```

## 总结

```sequence

producer->broker: sendMessageInTransaction()
broker->broker: backup RealTopic,RealQueueId to property\n setTopic(RMQ_SYS_TRANS_HALF_TOPIC)
broker->producer: SEND_OK
producer->producer: executeLocalTransaction()
broker-->>broker: TransactionalMessageCheckService\n check RMQ_SYS_TRANS_HALF_TOPIC & RMQ_SYS_TRANS_OP_HALF_TOPIC \n if isNeedCheck: resolveHalfMsg
broker-->>producer: sendCheckMessage
producer-->>producer: checkLocalTransaction\n COMMIT_MESSAGE\n ROLLBACK_MESSAGE\n UNKNOW
producer-->>broker: endTransactionOneway()
broker-->>broker: endTransaction()
producer->broker: endTransaction()
broker->broker: if commit: restore RealTopic,RealQueueId\n store RMQ_SYS_TRANS_OP_HALF_TOPIC\n if rollback: store RMQ_SYS_TRANS_OP_HALF_TOPIC
```

![RocketMQ-transaction](/upload/2019/1908-rocketmq.png)

## 问题

- 事务消息提交后, 消费失败怎么处理?
    > 业务方自己处理, RocketMQ没法处理
