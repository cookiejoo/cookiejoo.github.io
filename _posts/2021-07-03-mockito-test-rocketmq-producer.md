---
title: Mockito EventBus RocketMQ 生产者测试《第五章》
layout: post
coauthor: 曹德高
tags: [test, mockito]
categories: Unit Test
---

### 前言

上一章说到了RocketMQ在EvnetBus包装下的测试工具。本章主要讲MSF EventBus生产者操作。

### EventBus 生产者模拟测试

请看下面实例：

```java
import com.alibaba.fastjson.JSONObject;
import com.cdg.msf.event.bus.base.DelayTimeEnum;
import com.cdg.msf.event.bus.base.EventBus;
import com.cdg.msf.event.bus.base.MsfEventBusException;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.junit.Before;
import org.junit.Test;
import org.mockito.internal.util.reflection.FieldSetter;
import org.powermock.api.mockito.PowerMockito;

import java.io.UnsupportedEncodingException;
import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

/**
 * RocketMQ 生产者单元测试
 *
 * @author: Cookie.Joo
 * @create: 2020/12/01
 */
public class MsfEventBusProducerTest {

    MsfEventBusProducer msfEventBusProducer;

    @Before
    public void setUp() {
        msfEventBusProducer = mock(MsfEventBusProducer.class);
        DefaultMQProducer producer = mock(DefaultMQProducer.class);
        try {
            // 修改类中某个属性的方式：FieldSetter
            FieldSetter.setField(msfEventBusProducer, MsfEventBusProducer.class.getDeclaredField("applicationName"), "probe-app");
            FieldSetter.setField(msfEventBusProducer, MsfEventBusProducer.class.getDeclaredField("producer"), producer);

            SendResult sendResult = new SendResult();
            //调用一个参数的方法正常返回
            when(msfEventBusProducer.syncSend(any())).thenReturn(sendResult);
            //调用两个参数的方法抛异常
            doThrow(new MsfEventBusException()).when(msfEventBusProducer).syncSend(any(), any(String.class));
            
            when(msfEventBusProducer.syncSend(any(), any(String.class), any(Integer.class))).thenReturn(sendResult);
            when(msfEventBusProducer.syncSend(any(), any(String.class), any(Integer.class), any(DelayTimeEnum.class))).thenReturn(sendResult);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    @Test
    public void mqStartTest() {
        try {
            // 测试mq启动
            Method postConstruct = PowerMockito.method(MsfEventBusProducer.class, "init");
            postConstruct.invoke(msfEventBusProducer);
        } catch (Exception e) {
            assertThat(e.fillInStackTrace()).
                    isInstanceOfAny(MQClientException.class);
        }
    }

    @Test
    public void normalToSentMsgTest() {
        try {
            SendResult sendResult = msfEventBusProducer.syncSend(getEventBus());
            assertThat(sendResult).as("SendResult is not null").isNotNull();
        } catch (Exception e) {
            assertThat(e).as("SendRes No Exception").isNull();
        }
    }

    @Test
    public void producerSentMsgExceptionTest() {
        try {
            SendResult sendResult = msfEventBusProducer.syncSend(getEventBus(), "My Topic");
            assertThat(sendResult).as("SendResult is null").isNull();
        } catch (Exception e) {
            assertThat(e.fillInStackTrace()).
                    isInstanceOfAny(MsfEventBusException.class, UnsupportedEncodingException.class);
        }
    }

    private EventBus getEventBus() {
        EventBus eventBus = new EventBus();
        eventBus.setBizNo("123");
        eventBus.setPayload(new JSONObject());
        eventBus.setModuleNo("2323");
        eventBus.setRequestNo("123123123");
        eventBus.setEventTag("*");
        return eventBus;
    }
}
```

### 总结

1. 修改某个类的属性请用FieldSetter，要注意的是Mockito对static、final修饰属性是不支持的。

本章分享结束。