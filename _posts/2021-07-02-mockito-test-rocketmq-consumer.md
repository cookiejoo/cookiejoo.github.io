---
title: Mockito EventBus RocketMQ 消费者测试《第四章》
layout: post
coauthor: 曹德高
tags: [test, mockito]
categories: Unit Test
---

### 前言

在消息中间件中我们要模拟的是收发信息的动作，MQ这类的是依赖第三方消息中间件的，除去启动消息中间件去消费消息这一动作外，其实我们最关心的是我们接受到消息后怎么处理的问题，那么收消息是MQ中多线程去拉取数据的，也是官方jar提供工具类帮我们做的，所以我们需要入手的就是模拟有人给我们发消息，这个接收类是我们重点Mock的对象。本章主要讲如何在单元测试中模拟收RocketMQ服务器消息操作。

还是一如既往的不使用真正的服务器，直接用mock挡板。

### EventBus 消费者模拟测试

MSF框架中提供了抽象类，用户只要继承该类去实现handle方法即可，请看下面实例：

首先需要建一个子类的实现类，并且实现父类的handle方法

```java
import com.qihoo.finance.msf.event.bus.base.EventBus;

/**
 * @author: Cookie.Joo
 * @create: 2020/12/02
 */
public class ProbeMsfEventBusConsumer extends MsfEventBusConsumer {
    @Override
    public boolean handle(EventBus eventBus) {
        System.out.println("处理eventBus:" + eventBus);
        return true;
    }

    @Override
    public String getTopic() {
        return "msfEventBusDefaultTopic";
    }

    @Override
    public String getEventTag() {
        return "*";
    }
}
```

简单打印EventBus即可，下面是Mock Test的实例

```java
import com.google.common.collect.Lists;
import com.qihoo.finance.msf.event.bus.base.EventBus;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.common.consumer.ConsumeFromWhere;
import org.apache.rocketmq.common.message.MessageExt;
import org.apache.rocketmq.common.protocol.heartbeat.MessageModel;
import org.junit.Before;
import org.junit.Test;
import org.powermock.api.mockito.PowerMockito;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

/**
 * RocketMQ 消费者单元测试
 *
 * @author: Cookie.Joo
 * @create: 2020/12/01
 */
public class MsfEventBusConsumerTest {
    // 消费者线程
    ProbeMsfEventBusConsumer consumer;
    // 监听执行类
    ProbeMsfEventBusConsumer.DefaultMessageListenerConcurrently concurrently;

    @Before
    public void setUp() {
        consumer = mock(ProbeMsfEventBusConsumer.class);
        concurrently = consumer.new DefaultMessageListenerConcurrently();
        when(consumer.getConsumerGroupName()).thenReturn("com-app_event-bus_msfEventBusConsumerGroup");
        when(consumer.getPullBatchSize()).thenReturn(32);
        when(consumer.getTopic()).thenReturn("msfEventBusDefaultTopic");
        when(consumer.getEventTag()).thenReturn("*");
        when(consumer.getMessageModel()).thenReturn(MessageModel.CLUSTERING);
        when(consumer.getConsumeFromWhere()).thenReturn(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);

        when(consumer.getConsumeThreadMin()).thenReturn(20);
        when(consumer.getConsumeThreadMax()).thenReturn(64);

        // 注意：由于抽象类里面要模拟调用子类方法，这里调用子类handle的时候要去调用真实的子类实现，所以要配置这一句才能达到目标
        when(consumer.handle(any(EventBus.class))).thenCallRealMethod();
    }

    @Test
    public void mqStartTest() {
        try {
            // 测试mq启动
            Method postConstruct = PowerMockito.method(ProbeMsfEventBusConsumer.class, "startRocketMQPushConsumer");
            postConstruct.invoke(consumer);
        } catch (Exception e) {
            //解密失败
            assertThat(e.fillInStackTrace()).
                    isInstanceOfAny(MQClientException.class);
        }
    }

    @Test
    public void consumerTest() {
        // 测试消费消息逻辑
        MessageExt msg = new MessageExt();
        msg.setMsgId("123");
        msg.setBody("{}".getBytes());
        concurrently.consumeMessage(Lists.newArrayList(msg), null);
    }
}
```

请看执行的打印结果，启动MQ start的时候打印了开始消费日志；消费逻辑我用了多线程每5秒钟模拟收到一条日志，本例子已经去掉，这里用多线程并无意义，应该直接关注handle的实现逻辑。

![image-20201202142744053](/images/mockito-test-rocketmq-consumer/image-20201202142744053.png)

### 总结

1. 如多线程在@Test单元测试需要额外加CountDownLatch.await()守护线程，否则主线程执行完了，会立即System.exit(0)退出。
2. Mock父类抽象方法需要在执行实现子类的预备里面加thenCallRealMethod回调到真实实现方法。
3. 模拟MQ接收到处理这一环节是类中套public子类的语法，如concurrently = consumer.new DefaultMessageListenerConcurrently();这中new的方式使用上比较少，应当额外注意。

本章分享结束。