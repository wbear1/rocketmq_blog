# 先来个HelloWorld

RocketMQ是一个开源的分布式消息队列中间件。官网介绍：https://rocketmq.apache.org/
依照惯例，先通过HelloWorld程序也简单了解下RocketMQ的基本用法和基本概念，在之后的文章再详细介绍HelloWorld程序背后的原理和实现。对于MQ而言，HelloWorld程序主要是消息发送方（provider）和消息接收方（consumer）的代码实现，在RocketMQ中，不论是provider还是consumer都支持多种模式，下面分别一一介绍。

- [1、消息Provider](#1)
  - [同步发送](#1.1)
  - [异步发送](#1.2)
  - [任性发送](#1.3)
- [2、消息Consumer](#2)
  - [拉模式PullConsumer](#2.1)
  - [推模式PushConsumer](#2.1)
- [3、一些概念](#3)

<a name="1"></a> 
#### 1、消息Provider

<a name="1.1"></a>
##### 同步发送

RocketMQ的同步发送消息提供了多个方法，主要分为4类：1）直接发送mesage；2）发到指定messageQueue；3）发送message时，执行自定义的selector来选择发到哪个messageQueue; 4）发送批量消息。针对这4类方法，RocketMQ为每类方法还提供了指定超时时间的重载方法。

```java
/**
 * Send message in synchronous mode. This method returns only when the sending procedure totally completes.
 * </p>
 *
 * <strong>Warn:</strong> this method has internal retry-mechanism, that is, internal implementation will retry
 * {@link #retryTimesWhenSendFailed} times before claiming failure. As a result, multiple messages may potentially
 * delivered to broker(s). It's up to the application developers to resolve potential duplication issue.
 *
 * @param msg Message to send.
 * @return {@link SendResult} instance to inform senders details of the deliverable, say Message ID of the message,
 * {@link SendStatus} indicating broker storage/replication status, message queue sent to, etc.
 * @throws MQClientException if there is any client error.
 * @throws RemotingException if there is any network-tier error.
 * @throws MQBrokerException if there is any error with broker.
 * @throws InterruptedException if the sending thread is interrupted.
 */
SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException;

/**
 * Same to {@link #send(Message)} with target message queue specified in addition.
 *
 * @param msg Message to send.
 * @param mq Target message queue.
 * @return {@link SendResult} instance to inform senders details of the deliverable, say Message ID of the message,
 * {@link SendStatus} indicating broker storage/replication status, message queue sent to, etc.
 * @throws MQClientException if there is any client error.
 * @throws RemotingException if there is any network-tier error.
 * @throws MQBrokerException if there is any error with broker.
 * @throws InterruptedException if the sending thread is interrupted.
 */
SendResult send(Message msg, MessageQueue mq) throws MQClientException, RemotingException, MQBrokerException, InterruptedException;

/**
 * Same to {@link #send(Message)} with message queue selector specified.
 *
 * @param msg Message to send.
 * @param selector Message queue selector, through which we get target message queue to deliver message to.
 * @param arg Argument to work along with message queue selector.
 * @return {@link SendResult} instance to inform senders details of the deliverable, say Message ID of the message,
 * {@link SendStatus} indicating broker storage/replication status, message queue sent to, etc.
 * @throws MQClientException if there is any client error.
 * @throws RemotingException if there is any network-tier error.
 * @throws MQBrokerException if there is any error with broker.
 * @throws InterruptedException if the sending thread is interrupted.
 */
SendResult send(Message msg, MessageQueueSelector selector, Object arg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException;

SendResult send(final Collection<Message> msgs) throws MQClientException, RemotingException, MQBrokerException, InterruptedException;
```
下面是同步发送message时，执行自定义的selector的示例：
```java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.MessageQueueSelector;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageQueue;

import java.util.List;

public class SyncProducer {
    public static void main(String[] args) throws Exception {
        final String topic = "TopicTest";
        final String msgStr = "Hello,World";
        //创建MQProducer，指定其标识
        final DefaultMQProducer producer = new DefaultMQProducer("rocketmq-test-sync-producer");
        producer.setNamesrvAddr("172.0.0.1:9876"); //要连接RocketMQ broker，需要设定namesrv的地址
        producer.start();

        Message msg = new Message(topic, null, msgStr.getBytes("UTF-8"));
        final MessageQueueSelector selector = new MessageQueueSelector() {
            @Override
            public MessageQueue select(List<MessageQueue> mqs, Message msg, Object key) {
                int queueId = ((int) key) % mqs.size(); //根据key求余
                return mqs.get(queueId);
            }
        };

        for (int i = 0; i < 10; i++) {
            //这里以i作为选择的messageQueue的key，实际应用中可以根据业务来设计key
            producer.send(msg, selector, i);
        }

        producer.shutdown();
    }
}
```

<a name="1.2"></a>
##### 异步发送

异步发送的方法与同步发送类似，只是多了一个回调参数
```java
/**
 * Send message to broker asynchronously.
 * </p>
 *
 * This method returns immediately. On sending completion, <code>sendCallback</code> will be executed.
 * </p>
 *
 * Similar to {@link #send(Message)}, internal implementation would potentially retry up to
 * {@link #retryTimesWhenSendAsyncFailed} times before claiming sending failure, which may yield message duplication
 * and application developers are the one to resolve this potential issue.
 *
 * @param msg Message to send.
 * @param sendCallback Callback to execute on sending completed, either successful or unsuccessful.
 * @throws MQClientException if there is any client error.
 * @throws RemotingException if there is any network-tier error.
 * @throws InterruptedException if the sending thread is interrupted.
 */
public void send(Message msg, SendCallback sendCallback) throws MQClientException, RemotingException, InterruptedException;

/**
 * Same to {@link #send(Message, SendCallback)} with target message queue specified.
 *
 * @param msg Message to send.
 * @param mq Target message queue.
 * @param sendCallback Callback to execute on sending completed, either successful or unsuccessful.
 * @throws MQClientException if there is any client error.
 * @throws RemotingException if there is any network-tier error.
 * @throws InterruptedException if the sending thread is interrupted.
 */
public void send(Message msg, MessageQueue mq, SendCallback sendCallback) throws MQClientException, RemotingException, InterruptedException;

/**
 * Same to {@link #send(Message, SendCallback)} with message queue selector specified.
 *
 * @param msg Message to send.
 * @param selector Message selector through which to get target message queue.
 * @param arg Argument used along with message queue selector.
 * @param sendCallback callback to execute on sending completion.
 * @throws MQClientException if there is any client error.
 * @throws RemotingException if there is any network-tier error.
 * @throws InterruptedException if the sending thread is interrupted.
 */
public void send(Message msg, MessageQueueSelector selector, Object arg, SendCallback sendCallback) throws MQClientException, RemotingException, InterruptedException;

```

异步发送消息的示例如下所示：
```java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.MessageQueueSelector;
import org.apache.rocketmq.client.producer.SendCallback;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageQueue;

import java.util.List;
import java.util.concurrent.CountDownLatch;

public class AsyncProducer {
    public static void main(String[] args) throws Exception {
        final String topic = "TopicTest";
        final String msgStr = "Hello,World";
        //创建MQProducer，指定其标识
        final DefaultMQProducer producer = new DefaultMQProducer("rocketmq-test-async-producer");
        producer.setNamesrvAddr("172.0.0.1:9876"); //要连接RocketMQ broker，需要设定namesrv的地址
        producer.start();

        final Message msg = new Message(topic, null, msgStr.getBytes("UTF-8"));
        final MessageQueueSelector selector = new MessageQueueSelector() {
            @Override
            public MessageQueue select(List<MessageQueue> mqs, Message msg, Object key) {
                int queueId = ((int) key) % mqs.size(); //根据key求余
                return mqs.get(queueId);
            }
        };
        final CountDownLatch latch = new CountDownLatch(10);
        final SendCallback callback = new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                //这里可以根据sendResult来实现相应的业务逻辑
                System.out.println(sendResult);
                latch.countDown();
            }

            @Override
            public void onException(Throwable e) {

            }
        };

        for (int i = 0; i < 10; i++) {
            //这里以i作为选择的messageQueue的key，实际应用中可以根据业务来设计key
            producer.send(msg, selector, i, callback);
        }

        latch.await();
        producer.shutdown();
    }
}
```

<a name="1.3"></a>
##### 任性发送

这里的任性发送是指将调用send方法后，不关心发送结果，相当于不带callback的异步发送。同样地，也有三类方法：
```java
/**
 * Similar to <a href="https://en.wikipedia.org/wiki/User_Datagram_Protocol">UDP</a>, this method won't wait for
 * acknowledgement from broker before return. Obviously, it has maximums throughput yet potentials of message loss.
 *
 * @param msg Message to send.
 * @throws MQClientException if there is any client error.
 * @throws RemotingException if there is any network-tier error.
 * @throws InterruptedException if the sending thread is interrupted.
 */
public void sendOneway(Message msg) throws MQClientException, RemotingException, InterruptedException;

/**
 * Same to {@link #sendOneway(Message)} with target message queue specified.
 *
 * @param msg Message to send.
 * @param mq Target message queue.
 * @throws MQClientException if there is any client error.
 * @throws RemotingException if there is any network-tier error.
 * @throws InterruptedException if the sending thread is interrupted.
 */
public void sendOneway(Message msg, MessageQueue mq) throws MQClientException, RemotingException, InterruptedException;

/**
 * Same to {@link #sendOneway(Message)} with message queue selector specified.
 *
 * @param msg Message to send.
 * @param selector Message queue selector, through which to determine target message queue to deliver message
 * @param arg Argument used along with message queue selector.
 * @throws MQClientException if there is any client error.
 * @throws RemotingException if there is any network-tier error.
 * @throws InterruptedException if the sending thread is interrupted.
 */
public void sendOneway(Message msg, MessageQueueSelector selector, Object arg) throws MQClientException, RemotingException, InterruptedException;
```

<a name="2"></a>
#### 2、消息Consumer

<a name="2.1"></a>
##### 拉模式PullConsumer

拉模式PullConsumer，按字面意思理解就是client端主动发请求到服务端去拉取消息。主动拉取时，需要指定messageQueue，从哪里开始，以及最大拉取消息数量，同时对于已经消费的消息，还需要记录其offset。和推送模式相比，相对麻烦一些，下面示例其使用的大致流程，实际业务需要根据实际情况修改。
```java
import org.apache.rocketmq.client.consumer.DefaultMQPullConsumer;
import org.apache.rocketmq.client.consumer.MessageQueueListener;
import org.apache.rocketmq.client.consumer.PullResult;
import org.apache.rocketmq.client.consumer.PullStatus;
import org.apache.rocketmq.client.exception.MQBrokerException;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.common.message.MessageExt;
import org.apache.rocketmq.common.message.MessageQueue;
import org.apache.rocketmq.remoting.exception.RemotingException;

import java.util.List;
import java.util.Set;

public class PullConsumer {

    public static void main(String[] args) throws MQClientException, InterruptedException, RemotingException, MQBrokerException {
        final String topic = "TopicTest";
        DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("testConsumerGroup");
        consumer.setNamesrvAddr("172.0.0.1:9876");
        consumer.registerMessageQueueListener(topic, new MessageQueueListener() {
            @Override
            public void messageQueueChanged(String topic, Set<MessageQueue> mqAll, Set<MessageQueue> mqDivided) {
                //监听到rebalance之后的处理
            }
        });
        consumer.start();
        long startOffset = 0L;              //每次主动拉取消息的偏移量，需要自己维护
        int maxMessageNum = 100;            //每次拉取消息的最大数量
        String brokerName = "broker-0";     //指定messageQueue需要选取broker，可以通过接口获取
        int queueId = 0;                    //指定queueId
        while (true) {
            PullResult result = consumer.pull(new MessageQueue(topic, brokerName, queueId), "*", startOffset, maxMessageNum);
            PullStatus status = result.getPullStatus();
            switch (status) {
                case FOUND:
                    List<MessageExt> msgs = result.getMsgFoundList();
                    //对消息进行处理，处理完还要更新offset，consumer.updateConsumeOffset
                    break;
                case NO_NEW_MSG:
                case NO_MATCHED_MSG:
                    //可以考虑休息一下
                    break;
                case OFFSET_ILLEGAL:
                    //offset为啥会不合法呢，是不是offset的存储出问题了呢，需要自己处理
                    break;
                default:
                    break;
            }
        }
    }
}
```

<a name="2.2"></a>
##### 推模式PushConsumer

推模式PushConsumer则是服务端"主动"将消息推送给client端，通常服务端主动推送消息，会对服务端造成较大压力，因此RocketMQ只是在行为上看起来是实现了推送模式，底层实现依然是拉取。
```java
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeOrderlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeOrderlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerOrderly;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.common.consumer.ConsumeFromWhere;
import org.apache.rocketmq.common.message.MessageExt;

import java.util.List;
import java.util.concurrent.TimeUnit;

public class PushConsumer {

    public static void main(String[] args) throws MQClientException, InterruptedException {
        final String topic = "TopicTest";
        final DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("testConsumerGroup");
        consumer.setNamesrvAddr("172.0.0.1:9876");
        consumer.subscribe(topic, "*"); //第二个参数是对tag进行过滤的表达式，* 表示不过滤
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);
        //注册消息监听处理器，MessageListenerOrderly表示同一个messageQueue中的消息按顺序进行处理
        //如果不关心消息的处理顺序，这里可以注册MessageListenerConcurrently接口的实现
        consumer.registerMessageListener(new MessageListenerOrderly() {
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
                //msgs即是推送过来的消息，按实际业务来处理
                return ConsumeOrderlyStatus.SUCCESS;
            }
        });
        consumer.start();
        TimeUnit.MINUTES.sleep(10); //用于测试，只消费10分钟
        consumer.shutdown();
    }
}
```

<a name="3"></a>
#### 3、一些概念
