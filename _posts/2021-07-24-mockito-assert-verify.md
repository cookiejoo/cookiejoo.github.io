---
title: Mockito 断言和校验器《第九章》
layout: post
coauthor: 曹德高
tags: [test, mockito]
categories: Unit Test
---

### 前言

我们有很多篇幅介绍了如果使用mock挡板等工具做无外部依赖单元测试，本章聊一下单元测试中的断言和校验器。

### 断言Assertj

推荐用Assertj 断言，反观Assert不是那么的直观好用。Assertj支持字符串、数字、日期、List、Map、Class等类型，此外还提供了好用的 fail 方法。除此之外，对Java中的Exception、Iterable、JodaTime、Guava等都提供支持。

```java
    import static org.assertj.core.api.Assertions.*;

		@Test
    public void testList() {
        List<String> names = Lists.newArrayList("zhangsan", "lisi", "wangwu", "zhaoliu");
        // assertj 3 需要加 asList() 你可以一起校验也可以分开校验
        assertThat(names).asList().hasSize(4);
        // 是否有4个元素、开始的第一个是zhangsan、最后一个是zhaoliu、其中有lisi、只有一个wangwu
        assertThat(names).asList()
                .hasSize(4).startsWith("zhangsan").endsWith("zhaoliu")
                .contains("lisi", atIndex(1))
                .containsOnlyOnce("wangwu");
    }

    @Test
    public void testMap() {
        Map<String, String> map = Maps.newHashMap();
        map.put("a", "A");
        map.put("b", "B");
        map.put("c", "C");

        //提取extracting（keys）中的值，是否包含一个A值、是否没有包含D值
        assertThat(map).extracting("a", "b", "c").contains("A").doesNotContain("D");
        //map满足satisfies 一个b的key、b中的值是B
        assertThat(map).satisfies(s -> s.containsKey("b")).extracting("b").contains("B");
    }

    @Test
    public void testClass() {
        // 断言 没有注解类
        assertThat(PersonInfo.class).isNotAnnotation();
        // 断言 有注解类
        assertThat(Deprecated.class).isAnnotation();
        // 断言 存在注解为@Deprecated
        assertThat(PersonInfo.class).hasAnnotation(Deprecated.class);
        // 断言 不是接口
        assertThat(PersonInfo.class).isNotInterface();
        // 断言 Object 类是 PersonInfo 类的父类
        assertThat(Object.class).isAssignableFrom(PersonInfo.class);
    }

		@Test
    public void testContent() {
      	//一般用法就是比较内容的：assertThat(比较的内容).as(失败时的说明).xxx(结果)
      	boolean flag = true;
        assertThat(flag).as("flag is true.").isEqualTo(true);
    }

    //异常情况请看第七章有很多例子。
    @Test
    public void testException() {
        assertThatExceptionOfType(IOException.class).isThrownBy(() -> { throw new IOException("xxx!"); })
                .withMessage("xxx!")
                .withMessageContaining("xx")
                .withMessage("%s!", "xxx")
                .withStackTraceContaining("IOException")
                .withNoCause();
    }

		@Test
    public void testFail() {
      	//fail相当于抛异常，抛AssertionError的异常
        try {
            fail("在不检查任何条件的情况下使断言失败。显示一则消息");
        } catch (AssertionError ae) {
            System.out.println(ae);// 会有输出
        }
        try {
            failBecauseExceptionWasNotThrown(RuntimeException.class);
        } catch (AssertionError ae) {
            System.out.println(ae);// 会有输出
        }
    }
```

### 校验器Verify

校验器mockito和powerMockito合用

**被测试的方法**

```java
    @Resource
    private ArchTopologyNebulaDAO archTopologyNebulaDAO;

    @Resource
    private SkyWalkingManager skyWalkingManager;
		@Override
    public boolean execute(List<String> list) {
        if (!CollectionUtils.isEmpty(list)) {
            for (String str : list) {
              	//验证点3
                SwTopologyDTO data = skyWalkingManager.getServiceTopology(str);
                if (data != null) {
                    Set<String> nGqlsByDatabases = Sets.newHashSet();
                  	//验证点1
                    Map<String, String> nebulaDatabases = archTopologyNebulaDAO.query2MapBy("id", "type", str);
                    Map<String, SwTopologyDTO.Node> swNodeMap = Maps.newHashMap();
                    for (SwTopologyDTO.Node n : data.getNodes()) {
                        nGqlsByDatabases.add(n.getNodeId + str);
                    }
                  	//验证点2
                    archTopologyNebulaDAO.save2nebula(nGqlsByDatabases);
                }
            }
        }
        return true;
    }
```

**测试这么写**

```java
    @Test
    public void selectExecuteTest() throws NoSuchFieldException {
      	List<String> data = List<String> data = Lists.newArrayList("probe-app", "warehouse-app");

    		boolean flag = archTopologyService.execute(data);
				//校验是否被调用过2次
        verify(archTopologyNebulaDAO, times(2)).query2MapBy(any(), any(), any());
        verify(archTopologyNebulaDAO, times(2)).save2nebula(anyCollection());
				
      	//验证调用顺序
      	InOrder inOrder = inOrder(skyWalkingManager, archTopologyNebulaDAO);
        inOrder.verify(skyWalkingManager).getServiceTopology("probe-app");
        inOrder.verify(skyWalkingManager).getServiceTopology("warehouse-app");
      	
        assertThat(flag).as("flag is true.").isEqualTo(true);
    }
```

**参数匹配器**

在某些场景中，不光要对方法的返回值和调用进行验证，同时需要验证一系列交互后所传入方法的参数。那么我们可以用参数捕获器来捕获传入方法的参数进行验证，看它是否符合我们的要求。

```java
	argument.capture() 捕获方法参数
	argument.getValue() 获取方法参数值，如果方法进行了多次调用，它将返回最后一个参数值
	argument.getAllValues() 方法进行多次调用后，返回多个参数值


		@Test
    public void selectExecuteTest() throws NoSuchFieldException {
      	List<String> data = List<String> data = Lists.newArrayList("probe-app", "warehouse-app");

    		boolean flag = archTopologyService.execute(data);
  			
  			//验证save2nebula这个方法被调用4次，最后一次的参数个数是1
        ArgumentCaptor<List> argument = ArgumentCaptor.forClass(List.class);
        verify(archTopologyNebulaDAO, times(4)).save2nebula(argument.capture());
  			assertThat(argument.getValue().size()).as("size is 1").isEqualTo(1);
        assertThat(argument.getAllValues().size()).as("size is 4").isEqualTo(4);
		}
```

**额外说明**

```java
//any和anyString这种是有区别的，any代表包含null，anyString是不包含空的。如果是数组就这么用，比如
String write(byte[] fileByte)
any(byte[].class)
  
String doSomeThing(List<T> list)
any(List.class)
```

**静态方法验证**

静态方法验证需要用到PowerMockito，单单Mockito是无法满足的。而且静态方法测试有特殊的写法需要注意。

比如我在写sentinel的时候要用到里面的静态类帮我做拦截，我想知道ContextUtil.enter这个静态方法被调用了多少次，该怎么做

```java
		public boolean flowControl(EventBus eventBus) throws BlockException {
        if (!sentinelSwitch) {
            return handle(eventBus);
        }
        Entry entry = null;
        try {
          	//测试点
            ContextUtil.enter(sentinelResourceName());
            entry = SphU.entry(sentinelResourceName(), EntryType.IN);
            return handle(eventBus);
        } catch (BlockException ex) {
            throw ex;
        } finally {
            if (entry != null) {
                entry.exit();
            }
            ContextUtil.exit();
        }
    }

		public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> messageExtList) {
      	for (MessageExt msg : messageExtList) {
        		reconsumeTimes = msg.getReconsumeTimes();
        		String body = new String(msg.getBody());
        		EventBus eventBus = JSON.parseObject(body, EventBus.class);
        		boolean result = flowControl(eventBus);
      	}
    }
```

**测试这么写**

```java
		@Test
    public void consumeMessageNormalTest() throws NoSuchFieldException {
      	//静态mock在《第六章》也做了示例，请到里面查看。
        PowerMockito.mockStatic(ContextUtil.class);

        // 测试消费消息逻辑，模拟被调用了flowControl被调用了10次
        MessageExt msg = new MessageExt();
        msg.setMsgId("123");
        msg.setBody("{}".getBytes());

        for (int i = 0; i < 10; i++) {
            concurrently.consumeMessage(Lists.newArrayList(msg), null);
        }
				
      	//这里用PowerMockito的verifyStatic来校验
        PowerMockito.verifyStatic(ContextUtil.class, times(10));
        //重要：告知最后要verifyStatic验证的静态方法是哪一个，否则异常
        ContextUtil.enter(anyString());
				
    }
```

### 总结

基本上已经做了非常多常见场景的示例，学完已经能解决绝大部分的单元测试，单元测试的很多方法很多用法我们可以在使用中去发现，后期如果有更多好玩的方法我也会再写文章分享出来。
