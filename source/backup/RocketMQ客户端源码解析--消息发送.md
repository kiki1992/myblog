## 消息同步发送

### 发送配置

***发送超时时间***

DefaultMQProducer支持设置消息发送超时时间sendMsgTimeout，默认时间为3秒

```java
/**
 * DEFAULT SYNC -------------------------------------------------------
 */
public SendResult send(
    Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return send(msg, this.defaultMQProducer.getSendMsgTimeout());
}
```

```java
/**
 * Timeout for sending messages.
 */
private int sendMsgTimeout = 3000;
```

另外，也可以直接调用带超时参数的同步发送方法。

```java
@Override
public SendResult send(Message msg,
    long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.defaultMQProducerImpl.send(msg, timeout);
}
```

***发送失败是否重试***

DefaultMQProducer可以设置是否打开失败重试功能，默认关闭

```java
/**
 * Indicate whether to retry another broker on sending failure internally.
 */
private boolean retryAnotherBrokerWhenNotStoreOK = false;
```





## 发送共通逻辑

不管是同步，异步还是一次性消息，都调用了共通的消息发送方法，下面是方法的主要逻辑：

1.检查客户端状态

发送消息前需要确保客户端处于服务状态，否则消息无法发送。

```java
private void makeSureStateOK() throws MQClientException {
    if (this.serviceState != ServiceState.RUNNING) {
        throw new MQClientException("The producer service state not OK, "
            + this.serviceState
            + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
            null);
    }
}
```

2.检查消息合法性

发送消息前还需要检查消息的合法性，主要检查发送Topic以及消息body的合法性。

***这里会检查消息body长度，可以通过配置限制客户端可发送消息的长度***

```java
public static void checkMessage(Message msg, DefaultMQProducer defaultMQProducer)
    throws MQClientException {
    if (null == msg) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message is null");
    }
    // topic
    Validators.checkTopic(msg.getTopic());

    // body
    if (null == msg.getBody()) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message body is null");
    }

    if (0 == msg.getBody().length) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message body length is zero");
    }

    if (msg.getBody().length > defaultMQProducer.getMaxMessageSize()) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL,
            "the message body size over max value, MAX: " + defaultMQProducer.getMaxMessageSize());
    }
}
```

3.检查消息Topic发布信息（TODO 发布信息拉取逻辑）

下一步检查消息的Topic发布信息，如果发布信息不存在或者不可用，则从NameServer拉取最新的发布信息。

4.消息发送（同步发送包含重试）

在确认发送环境之后，将根据发送类型确定消息发送次数，并执行发送操作。

***确认发送次数***

```java
// 同步发送包含失败重试
int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
```

***执行发送操作***

```java
for (; times < timesTotal; times++) {
    String lastBrokerName = null == mq ? null : mq.getBrokerName();
    // 首先选择一个队列来发送消息，如果容错策略开启，需要考虑容错机制对队列选择的影响
    MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
    if (mqSelected != null) {
        mq = mqSelected;
        brokersSent[times] = mq.getBrokerName(); // 记录发送的broker
        try {
            beginTimestampPrev = System.currentTimeMillis();
            // 执行消息发送操作
            sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);
            endTimestamp = System.currentTimeMillis();
            // 更新消息发送容错信息
            this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
            switch (communicationMode) {
                // 异步发送以及一次性发送的情况在执行消息发送操作并更新容错信息后立即返回
                case ASYNC:
                    return null;
                case ONEWAY:
                    return null;
                case SYNC:
                    // 同步发送时失败重发开启则尝试重新发送
                    if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                        if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                            continue;
                        }
                    }

                    return sendResult;
                default:
                    break;
            }
        // 发送失败处理，代码省略
}
```