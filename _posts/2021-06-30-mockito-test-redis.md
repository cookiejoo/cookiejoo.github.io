---
title: Mockito Redis测试《第二章》
layout: post
coauthor: 曹德高
tags: [test, mockito]
categories: Unit Test
---

### 前言

上一章说到了Mockito测试工具，大概入门的使用了该工具模拟调用Dubbo接口。本章主要讲如何在单元测试中模拟Redis服务器操作。

还是一如既往的不使用真正的服务器，直接用mock挡板。

### Redis模拟测试

请看下面实例：

```java

import com.cdg.msf.cache.redis.RedisCache;
import org.junit.Before;
import org.junit.Test;
import org.springframework.data.redis.connection.RedisConnection;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.*;

import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

/**
 * redis模板测试类
 *
 * @author: Cookie.Joo
 * @create: 2020/11/30
 */
public class RedisCacheTest {

    private RedisTemplate redisTemplate;

    private RedisCache redisCache;

    /**
     * mock掉整个RedisTemplate
     * 有需要其他的工具方法自行在附加即可
     */
    @Before
    public void setUp() {
        redisTemplate = mock(RedisTemplate.class);
        ValueOperations valueOperations = mock(ValueOperations.class);
        SetOperations setOperations = mock(SetOperations.class);
        HashOperations hashOperations = redisTemplate.opsForHash();
        ListOperations listOperations = redisTemplate.opsForList();
        ZSetOperations zSetOperations = redisTemplate.opsForZSet();
        when(redisTemplate.opsForSet()).thenReturn(setOperations);
        when(redisTemplate.opsForValue()).thenReturn(valueOperations);
        when(redisTemplate.opsForHash()).thenReturn(hashOperations);
        when(redisTemplate.opsForList()).thenReturn(listOperations);
        when(redisTemplate.opsForZSet()).thenReturn(zSetOperations);
        RedisOperations redisOperations = mock(RedisOperations.class);
        RedisConnection redisConnection = mock(RedisConnection.class);
        RedisConnectionFactory redisConnectionFactory = mock(RedisConnectionFactory.class);
        when(redisTemplate.getConnectionFactory()).thenReturn(redisConnectionFactory);
        when(valueOperations.getOperations()).thenReturn(redisOperations);
        when(redisTemplate.getConnectionFactory().getConnection()).thenReturn(redisConnection);

        // redis工具类初始化
        redisCache = mock(RedisCache.class);
        // 调用get方法返回固定值
        when(redisTemplate.opsForValue().get("caodegao")).thenReturn("good");

        doThrow(new RuntimeException()).when(redisCache).set("caodegao", "caodegao");
    }


    @Test
    public void setTest() {
        assertThatThrownBy(() -> redisCache.set("caodegao", "caodegao"))
                .isInstanceOf(RuntimeException.class);
    }

    @Test
    public void redisTemplateGetTest() {
        String value = redisTemplate.opsForValue().get("caodegao").toString();
        assertThat(value).as("校验get(caodegao)").isEqualTo("good");
    }

    @Test
    public void redisTemplateGet1Test() {
        Object value = redisTemplate.opsForValue().get("cookie.joo");
        assertThat(value).as("校验get(cookie.joo)").isNull();
    }
}
```

![image-20201201145426816](/images/mockito-test-redis/image-20201201145426816.png)

提示：我已经把打印预计换成assertThat断言，为了做个图，所以打印为了截图，真实环境请勿写打印语句。

本章分享结束。