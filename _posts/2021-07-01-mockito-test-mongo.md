---
title: Mockito MongoDB测试《第三章》
layout: post
coauthor: 曹德高
tags: [test, mockito]
categories: Unit Test
---

### 1：前言

上一章说到了Redis测试工具，我们需要模拟的是Redis连接挡板。本章主要讲如何在单元测试中模拟Mongo服务器操作。

还是一如既往的不使用真正的服务器，直接用mock挡板。

### 2：Mongo模拟测试

请看下面实例：

```java
import com.google.common.collect.Lists;
import com.mongodb.MongoClient;
import com.mongodb.client.ListDatabasesIterable;
import org.bson.Document;
import org.bson.types.ObjectId;
import org.junit.Before;
import org.junit.Test;
import org.mongodb.morphia.Datastore;
import org.mongodb.morphia.Key;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Query;

import java.util.List;

import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

/**
 * MongoDB工具测试
 *
 * @author: Cookie.Joo
 * @create: 2020/12/01
 */
public class MongoDBTest {

    private Datastore datastore;
    private MongoClient mongoClient;
    private MongoTemplate mongoTemplate;

    @Before
    public void setUp() {
        // Datastore mock
        datastore = mock(Datastore.class);

        Key<CreditTestEntity> key = new Key<>(CreditTestEntity.class, "CreditTestEntity", new ObjectId());
        when(datastore.save(any(CreditTestEntity.class))).thenReturn(key);
        CreditTestEntity one = new CreditTestEntity();
        one.setId(new ObjectId());
        when(datastore.get(any(CreditTestEntity.class))).thenReturn(one);

        // MongoClient mock
        mongoClient = mock(MongoClient.class);
        ListDatabasesIterable<Document> documents = mock(ListDatabasesIterable.class);

        when(mongoClient.listDatabases()).thenReturn(documents);

        // MongoTemplate mock
        mongoTemplate = mock(MongoTemplate.class);

        CreditTestEntity entity = new CreditTestEntity();
        entity.setId(new ObjectId());
        entity.setAppId("com-app");
        List<CreditTestEntity> list = Lists.newArrayList(entity);
        when(mongoTemplate.find(new Query(), CreditTestEntity.class)).thenReturn(list);
    }

    @Test
    public void saveTest() {
        CreditTestEntity entity = new CreditTestEntity();
        String id = datastore.save(entity).getId().toString();
        assertThat(id).as("校验datastore.save(entity)").isNotNull();
    }

    @Test
    public void getTest() {
        CreditTestEntity entity = datastore.get(new CreditTestEntity());
        assertThat(entity).as("校验datastore.get(entity)").isNotNull();
    }

    @Test
    public void listDatabasesTest() {
        ListDatabasesIterable<Document> documents = mongoClient.listDatabases();
        assertThat(documents).as("校验mongoClient.listDatabases()").isNotNull();
    }

    @Test
    public void findTest() {
        List<CreditTestEntity> list = mongoTemplate.find(new Query(), CreditTestEntity.class);
        assertThat(list).
                as("校验mongoTemplate.find()第一个元素appId的值是否是com-app").
                asList().
                hasSize(1).
                element(0).
                hasFieldOrPropertyWithValue("appId", "com-app");
    }
}
```

最后运行的结果

![image-20201201145854005](/images/mockito-test-mongo/image-20201201145854005.png)

### 3：总结

其实从第二章到本章第三章，其实都是千变一律的模拟一个第三方工具来测验的，可能会产生质疑，这会不会和实际上不一样。我个人认为并没有多大差别，连接任何第三方存储其实是连接工具的问题，我们并非测试连接工具，我们是优先测试我们的功能在某些特定的数据下是否能正常运行，if-else语句是否都覆盖整个流程，数据的准备落盘并非是我们的首要关系的。连接到第三方工具这一步骤我们要测试的并不频繁，连接顶多是调优第三方jar支持的参数调整，与关键业务流程没有太强烈关系。单元测试昨晚后，我们启动到我们的应用去再走一次完整流程，才是正确的测试方式。

本章分享结束。

