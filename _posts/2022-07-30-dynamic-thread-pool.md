---
title: 动态线程池设计
layout: post
author: 曹德高
tags: [java]
categories: Java
---

### 背景

我们平时使用线程池参数都是写死在代码中的，需要改变线程池参数则需重启应用才能有效(必须要配置化参数)，那么动态变更线程池则非常有效的解决普通线程池在调优方面的便利问题。

### 如何设计它？

在我们的第一反应就是将每一个线程池管理起来，则每一个线程池都必须有一个id唯一标识，再有则必须要有个存储来存放线程池id对应的参数数据，还需要有个控制台UI方便我们实时操作变更，并且能显示出变更后的执行结果，我还想看到每10秒钟该线程池的执行情况，并绘制出图标一目了然的知道在执行的线程数、已经执行完成的任务数。再高级一点的则是任务的执行耗时，并能够配置执行任务耗时告警，出现拒绝任务的数据到达多少时能告警出来。围绕这个猜想，我们画图设置一下。

![条件结构流程图](/images/2022-07-30-dynamic-thread-pool/2022-07-30-dynamic-thread-pool.png)

我们规划一下项目的脚手架

* **dynamic-threadpool-core:** 对应用的加载时进行类扫描，对线程池收集；
* **dynamic-threadpool-server:** 控制台web，对接线程池管理、展示界面；
* **dynamic-threadpool-common:** 一些两边都会用到的工具类和实体类等；
* **dynamic-threadpool-alam:** 告警功能模块；
* **dynamic-threadpool-discover:** 上报心跳、线程运行信息，接收线程池变化通知。

这样对应总体的设计原形已经完全满足了，接下来对功能进行填充完善。

### 线程池封装

在开发核心包的时候我就不停的在想，该如何有效的进行线程池ThreadPoolExecutor接口的管理？并且我们自家中间件的类中也能够使用我们这个功能，如果只是做一个新的线程池封装，不考虑其他的系统怎么用的话一股脑直接写一个新的封装类就好，如果考虑兼容其他系统自己用的线程池那么就要考虑怎么把他们写的线程池纳入进来适配。

当然，如果只是自己自High的写一个动态线程池写起来还是非常简单的，但是自High代表着小众，你也无法推广给别人，如果你觉得自己写的代码设计的系统很牛，那么整合现有的东西是必须要做的，要不然走不远，也不会得到支持。**这句话要仔细推敲！**

回到正题，管理线程池ThreadPoolExecutor接口，我们需要借助Spring AOP，把ThreadPoolExecutor接口扫描出来，那么必须要写一个封装类并且加一个注解，Spring的类扫描很容易让我们找到我们自己的封装类，加一个注解是为了把其他应用的线程池用，打上该注解的线程池则一并收集到我们的管理端。

**线程池管理封装**

```java
/**
 * 封装自家线程池包装管理类，多了两个属性->归属系统、线程池ID
 * 提供执行常用的执行方法execute、submit
 * 实现DisposableBean类用于重写与销毁线程池方法
 */
@Data
public class DynamicThreadPoolWrapper implements DisposableBean {

    private String serviceName; 
    private String threadPoolId;

    private ThreadPoolExecutor executor;

    public DynamicThreadPoolWrapper(String threadPoolId, ThreadPoolExecutor threadPoolExecutor) {
        this.threadPoolId = threadPoolId;
        this.executor = threadPoolExecutor;
    }

    public void execute(Runnable command) {
        executor.execute(command);
    }

    public Future<?> submit(Runnable task) {
        return executor.submit(task);
    }

    public <T> Future<T> submit(Callable<T> task) {
        return executor.submit(task);
    }

    @Override
    public void destroy() throws Exception {
        if (executor != null && executor instanceof AbstractDynamicExecutorSupport) {
            ((AbstractDynamicExecutorSupport) executor).destroy();
        }
    }
}

```

**线程池封装**

```java
/**
 * 自家线程封装类
 * 多了线程拒绝统计数、当个线程执行时间的计算
 */
@Getter
@Setter
public class DynamicThreadPoolExecutor extends ThreadPoolExecutor {

    private Long executeTimeOut;

    private TaskDecorator taskDecorator;

    private final String threadPoolId;

    private final AtomicLong rejectCount = new AtomicLong();

    private final ThreadLocal<Long> startTime = new ThreadLocal<>();

    public DynamicThreadPoolExecutor(int corePoolSize,
                                     int maximumPoolSize,
                                     long keepAliveTime,
                                     TimeUnit unit,
                                     long executeTimeOut,
                                     boolean waitForTasksToCompleteOnShutdown,
                                     long awaitTerminationMillis,
                                     @NonNull BlockingQueue<Runnable> workQueue,
                                     @NonNull String threadPoolId,
                                     @NonNull ThreadFactory threadFactory,
                                     @NonNull RejectedExecutionHandler handler) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, waitForTasksToCompleteOnShutdown, awaitTerminationMillis, workQueue, threadPoolId, threadFactory, handler);
        this.threadPoolId = threadPoolId;
        this.executeTimeOut = executeTimeOut;
        // 为什么要自己实现拒绝策略？
        RejectedExecutionHandler rejectedProxy = RejectedProxyUtil.createProxy(handler, rejectCount);
        // extends ThreadPoolExecutor类的方法，替换成自家实现的拒绝策略接口
        setRejectedExecutionHandler(rejectedProxy);
    }

    @Override
    public void execute(@NonNull Runnable command) {
        if (taskDecorator != null) {
            command = taskDecorator.decorate(command);
        }
        super.execute(command);
    }

    @Override
    protected void beforeExecute(Thread t, Runnable r) {
        if (executeTimeOut == null || executeTimeOut <= 0) {
            return;
        }
        this.startTime.set(System.currentTimeMillis());
    }
		// 计算单个线程执行耗时，可以做监控告警
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        if (executeTimeOut == null || executeTimeOut <= 0) {
            return;
        }
        try {
            long startTime = this.startTime.get();
            long endTime = System.currentTimeMillis();
            long executeTime = endTime - startTime;
            if(executeTimeOut != null && executeTime > executeTimeOut){
              	// 记录到该应用的事件中，异步上报到控制台
            		// 内容省略
            }
              
        } finally {
            this.startTime.remove();
        }
    }

  	// 获取拒绝任务个数
    public Long getRejectCountNum() {
        return rejectCount.get();
    }
}

```

该类有一个疑问，为什么要自己实现一个拒绝策略类？

因为我们需要知道拒绝策略在触发统计一下个数，原生的拒绝策略是没有办法提供这个功能的，所以这个功能必须我们来做，详细看代码注释。

```java
/**
 * 使用代理类从原生的类中进行增强
 */
public class RejectedProxyUtil {
    public static RejectedExecutionHandler createProxy(RejectedExecutionHandler rejectedExecutionHandler, AtomicLong rejectedNum) {
        RejectedExecutionHandler rejectedProxy = (RejectedExecutionHandler) Proxy
                .newProxyInstance(
                        rejectedExecutionHandler.getClass().getClassLoader(),
                        new Class[]{RejectedExecutionHandler.class},
                        new RejectedProxyInvocationHandler(rejectedExecutionHandler, rejectedNum));
        return rejectedProxy;
    }
}

@AllArgsConstructor
public class RejectedProxyInvocationHandler implements InvocationHandler {

    private final Object target;

    private final AtomicLong rejectCount;
		// 在执行原生方法时调用封装类的自有属性rejectCount去自增1，这样拒绝策略被触发时，我们就能统计到拒绝数了。
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        rejectCount.incrementAndGet();
        try {
            return method.invoke(target, args);
        } catch (InvocationTargetException ex) {
            throw ex.getCause();
        }
    }
}
```

基本的线程池封装已经完毕，我们看看如何用Spring收集线程池的Bean对象。（这里唠点题外话：我之前理解的动态线程池以为是可以动态把线程池new出来的，后来看到原生线程池ThreadPoolExecutor接口发现原来他提供了很多set参数的方法。这样我们在动态修改参数的时候非常方便了。）由于我们的线程池不是动态的new，我们动态线程池的规范是把new线程池放到Spring Bean里面产生的，那么我们就可以在系统启动的时候去扫描所有的类即可。换句话说其他应用使用的基于Spring架构开发的动态线程池必然是单独使用Bean管理的方式来创建线程池的。最后把线程池都收集到应用的内存中，用GlobalThreadPoolManage收集，没有什么特别的就是一个Map

```java
/**
 * 我们用BeanPostProcessor接口
 * Spring会在Bean创建的时候经过一系列的Processor后到达我们这个类的postProcessAfterInitialization方法
 */
@Slf4j
public class DynamicThreadPoolPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof DynamicThreadPoolExecutor) {
            DynamicThreadPool dynamicThreadPool;
            try {
              // 通过ApplicationContextHolder去找所有这个类的
              dynamicThreadPool = DynamicThreadPoolAnnotationUtil.findAnnotationOnBean(beanName, DynamicThreadPool.class);
              if (Objects.isNull(dynamicThreadPool)) {
                return bean;
              }
                
            } catch (Exception ex) {
                log.error("Failed to create dynamic thread pool in annotation mode.", ex);
                ex.printStackTrace();
                return bean;
            }
            DynamicThreadPoolExecutor dynamicExecutor = (DynamicThreadPoolExecutor) bean;
            DynamicThreadPoolWrapper wrap = new DynamicThreadPoolWrapper(dynamicExecutor.getThreadPoolId(), dynamicExecutor);
            ThreadPoolExecutor remoteExecutor = fillPoolAndRegister(wrap);
            return remoteExecutor;
        }
        if (bean instanceof DynamicThreadPoolWrapper) {
            DynamicThreadPoolWrapper wrap = (DynamicThreadPoolWrapper) bean;
            fillPoolAndRegister(wrap);
        }
        return bean;
    }

    /**
     * 初始化线程池和线程池存储到内存Map中
     */
    protected ThreadPoolExecutor fillPoolAndRegister(DynamicThreadPoolWrapper dynamicThreadPoolWrap) {
        String threadPoolId = dynamicThreadPoolWrap.getThreadPoolId();
        ThreadPoolExecutor newDynamicPoolExecutor = dynamicThreadPoolWrap.getExecutor();
        /**
         * 到远程端获取原本的配置并设置初始线程池参数
         * 系统启动后自动加载配置里面的配置进行初始化线程，这样可以保证启动时的线程池和管理端的配置一致
         */
        ExecutorProperties executorProperties = Http or File;

        if (executorProperties != null) {
            try {
                BlockingQueue workQueue = QueueTypeEnum.createBlockingQueue(executorProperties.getQueueType(), executorProperties.getQueueCapacity());
                String threadNamePrefix = executorProperties.getThreadNamePrefix();
                newDynamicPoolExecutor = ThreadPoolBuilder.builder()
                        .dynamicPool()
                        .workQueue(workQueue)
                        .threadFactory(StringUtils.isNotBlank(threadNamePrefix) ? threadNamePrefix : threadPoolId)
                        .executeTimeOut(Optional.ofNullable(executorProperties.getExecuteTimeOut()).orElse(0L))
                        .poolThreadSize(executorProperties.getCorePoolSize(), executorProperties.getMaximumPoolSize())
                        .keepAliveTime(executorProperties.getKeepAliveTime(), TimeUnit.SECONDS)
                        .rejected(RejectedTypeEnum.createPolicy(executorProperties.getRejectedHandler()))
                        .allowCoreThreadTimeOut(executorProperties.getAllowCoreThreadTimeOut())
                        .build();
            } catch (Exception ex) {
                log.error("Failed to initialize thread pool configuration. error :: {}", ex);
            } finally {
                if (Objects.isNull(dynamicThreadPoolWrap.getExecutor())) {
                    dynamicThreadPoolWrap.setExecutor(CommonDynamicThreadPool.getInstance(threadPoolId));
                }
                dynamicThreadPoolWrap.setInitFlag(Boolean.TRUE);
            }
        }

        dynamicThreadPoolWrap.setExecutor(newDynamicPoolExecutor);

      	// 存储到内存中Map<线程ID,线程池管理类>，Map<线程ID,线程池参数信息类>
        GlobalThreadPoolManage.registerPool(dynamicThreadPoolWrap.getThreadPoolId(), dynamicThreadPoolWrap);
        GlobalThreadPoolManage.registerPoolParameter(dynamicThreadPoolWrap.getThreadPoolId(), buildExecutorProperties(threadPoolId, newDynamicPoolExecutor));

        return newDynamicPoolExecutor;
    }
		/**
		 * ExecutorProperties就是一些参数字段，代码省略
		 */
    private ExecutorProperties buildExecutorProperties(String threadPoolId, ThreadPoolExecutor executor) {
        
        ExecutorProperties executorProperties = new ExecutorProperties();
        BlockingQueue<Runnable> queue = executor.getQueue();
        int queueSize = queue.size();
        String queueType = queue.getClass().getSimpleName();
        int remainingCapacity = queue.remainingCapacity();
        int queueCapacity = queueSize + remainingCapacity;
        executorProperties.setCorePoolSize(executor.getCorePoolSize())
                .setMaximumPoolSize(executor.getMaximumPoolSize())
                .setAllowCoreThreadTimeOut(executor.allowsCoreThreadTimeOut())
                .setKeepAliveTime(executor.getKeepAliveTime(TimeUnit.SECONDS))
                .setQueueType(queueType)
                .setQueueCapacity(queueCapacity)
                .setRejectedHandler(((DynamicThreadPoolExecutor) executor).getRedundancyHandler().getClass().getSimpleName())
                .setThreadPoolId(threadPoolId);
        return executorProperties;
    }
}

```

### 线程池应用服务发现

到这里core包的任务已经完成使命了，接下来要将本地的线程池上报到服务端，那么需要在server中实现上报接口，并且在服务端实现实时修改线程池参数方法，还需要提供应用端启动时拉取线程池的方法。

我推荐服务端主动去推送线程池参数，每隔几分钟推送一次这个应用的线程池配置，应用端判断如果没有变化就不更新到线程池即可，这样做的一个好处是服务端UI修改线程池参数，可能整体下发到应用端时某个节点暂时失联了无法更新到。还有应用重启时会主动拉取一次配置去初始化启动线程池，已经很严谨了。

应用先上报心跳信息，上报后服务端才会知道推送的应用端所有的节点和端口，discover包开发一个http服务调用和接收即可。上报心跳一分钟一次即可，服务端也做好长时间没有上报就下线或自动剔除动作。再上报自身GlobalThreadPoolManage.registerPool的线程池运行状态，每隔20秒就上报一次，服务端做好记录，我是用Apache的CircularFifoQueue固定队列只存储60个Metric就可以简单的绘制图表实时观看了，如需要长时间的就要实现存储到数据库。

```java
/**
 * 动态更新线程池方法
 * 把能原生能开放设置的参数使用上
 */
public class HttpRefresherHandler {

    public static void dynamicRefreshPool(String threadPoolId, ExecutorProperties properties) {
        ExecutorProperties beforeProperties = GlobalThreadPoolManage.getPoolParameter(properties.getThreadPoolId());
        ThreadPoolExecutor executor = GlobalThreadPoolManage.getExecutorService(threadPoolId).getExecutor();

        if (properties.getCorePoolSize() != null && !Objects.equals(properties.getCorePoolSize(),beforeProperties.getCorePoolSize())) {
            executor.setCorePoolSize(properties.getCorePoolSize());
        }

        if (properties.getMaximumPoolSize() != null && !Objects.equals(properties.getMaximumPoolSize(),beforeProperties.getMaximumPoolSize())) {
            executor.setMaximumPoolSize(properties.getMaximumPoolSize());
        }

        if (properties.getAllowCoreThreadTimeOut() != null) {
            executor.allowCoreThreadTimeOut(properties.getAllowCoreThreadTimeOut());
        }
      
      	// 线程超时水位值，非原生线程池的参数，就是咱们说的耗时告警用
        if (!Objects.equals(beforeProperties.getExecuteTimeOut(), properties.getExecuteTimeOut())) {
            if (executor instanceof AbstractDynamicExecutorSupport) {
                ((DynamicThreadPoolExecutor) executor).setExecuteTimeOut(properties.getExecuteTimeOut());
            }
        }
				// 如果拒绝策略有变化，也要用代理模式增强后设置进来，才有拒绝数的功能
        if (!Objects.equals(beforeProperties.getRejectedHandler(), properties.getRejectedHandler())) {
            RejectedExecutionHandler rejectedExecutionHandler = RejectedTypeEnum.createPolicy(properties.getRejectedHandler());
            if (executor instanceof AbstractDynamicExecutorSupport) {
                DynamicThreadPoolExecutor dynamicExecutor = (DynamicThreadPoolExecutor) executor;
                dynamicExecutor.setRedundancyHandler(rejectedExecutionHandler);
                AtomicLong rejectCount = dynamicExecutor.getRejectCount();
                rejectedExecutionHandler = RejectedProxyUtil.createProxy(rejectedExecutionHandler, rejectCount);
            }
            executor.setRejectedExecutionHandler(rejectedExecutionHandler);
        }

        if (properties.getKeepAliveTime() != null) {
            executor.setKeepAliveTime(properties.getKeepAliveTime(), TimeUnit.SECONDS);
        }

    }
}

```



### 线程池告警

alam包实现告警功能，对接应用和员工系统，下面这个类已经实现了执行耗时，超过了多少阈值可以记录到某个存储介质中，上报到server端即可，server包引用alam包，方便发送告警信息到制定的应用开发者，每个公司都有自己的告警im接口，有这个包去规划实现。

```java
/**
 * 回顾一下这个类，我们封装的这个线程池类就能获取到告警要用的信息
 */
public class DynamicThreadPoolExecutor{
    ......
		// 计算单个线程执行耗时，可以做监控告警
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        if (executeTimeOut == null || executeTimeOut <= 0) {
            return;
        }
        try {
            long startTime = this.startTime.get();
            long endTime = System.currentTimeMillis();
            long executeTime = endTime - startTime;
            if(executeTimeOut != null && executeTime > executeTimeOut){
              	// 记录到该应用的事件中，异步上报到控制台
            		// 内容省略
            }
              
        } finally {
            this.startTime.remove();
        }
    }
    ......
  	// 获取拒绝任务个数
    public Long getRejectCountNum() {
        return rejectCount.get();
    }
}
```

### 最后

最后就是UI了，自己实现一套UI去管理和展示即可，还有一些告警配置什么的，基本上动态线程池的功能设计就已经结束了，存储配置的选择由自身去做，用什么都可以，初步可以先用Map放内存，真正申请成项目后就可以申请资源去做更可靠的存储和完善更多的功能。

github上有很多动态线程池项目做的都很不错，我这个也是重复的轮子，上手后才知道很多东西的细节要思考很久才有较好的解决方式，并且参考别人的实现后，你有了对比，再总结融合成你自己的东西，这也是一种成长。经验都是经历过才能验证，动手才知深浅，我年纪轻轻就达到三千二一个月工资，早上都是吃两个牛肉包的，你敢想？所以说有钱真好，年轻人、加油！