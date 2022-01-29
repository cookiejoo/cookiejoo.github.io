---

title: "Spring Bean初始化流程"
layout: post
author: 曹德高
tags: [spring]
categories: Spring
---

### **前言**

对于Bean的初始化流程，我觉得无需太专注去了解的，因为里面的东西太多了，对我们有啥帮助吗？没有，因为你只是应付将来可能面试时别人会问而已，拿到Offer了基本就扔掉了，如果是我的话我就直接说忘了或没怎么关注，我反而觉得里面的一些设计模式需要去探索一下的，远比应付面试要好千倍万倍。

我这里也随便简单过一下，想到哪就聊到哪吧。我想鼓励你的是：没关注就是没关注，如果很重要我们就去补习，不是每一个人都有过目不忘的本领，我们时刻保持一颗学习的心态去面对工作和生活就好了。

### 示例

下面我把关键的类方法贴出来大概说说它的流程是干嘛的，如果不感兴趣就瞎过一遍就行，重要的是里面的具体实现是值得我们去学习和深究的，我也是看了几遍源码去理解和找一些设计模式思想方法学习的，反正我也记不住，我就记住关键的一点什么时候创建Bean的。

org.springframework.context.support.AbstractApplicationContext#refresh

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized(this.startupShutdownMonitor) {
      //容器刷新前的准备，设置上下文状态，获取属性，验证必要的属性等
        this.prepareRefresh();
      //获取新的beanFactory，销毁原有beanFactory、为每个bean生成BeanDefinition
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
      //配置标准的beanFactory，设置ClassLoader，设置SpEL表达式解析器，添加忽略注入的接口，添加bean，添加bean后置处理器等
        this.prepareBeanFactory(beanFactory);

        try {
          //模板方法，此时，所有的beanDefinition已经加载，但是还没有实例化。
					//允许在子类中对beanFactory进行扩展处理。比如添加ware相关接口自动装配设置，添加后置处理器等，是子类扩展
            this.postProcessBeanFactory(beanFactory);
          //实例化并调用所有注册的beanFactory后置处理器
            this.invokeBeanFactoryPostProcessors(beanFactory);
          //实例化和注册beanFactory中扩展了BeanPostProcessor的bean
          //如@Autowired、@Required、@PreDestroy、@PostConstruct、@Resource等注解
            this.registerBeanPostProcessors(beanFactory);
          //初始化国际化工具类
            this.initMessageSource();
          //初始化事件广播器
            this.initApplicationEventMulticaster();
          //模板方法，在容器刷新的时候可以自定义逻辑，不同的Spring容器做不同的事情
            this.onRefresh();
          //注册监听器，广播early application events
            this.registerListeners();
          //实例化所有剩余的（非懒加载）单例
          //**这步很重要了，上面都是做一些准备工作，这里就要开始创建Bean了，和如何解决循环依赖的问题都在这里面去触发调用，具体请看另外一篇文件，这里是触发创建Bean，所以要额外提示一下**
            this.finishBeanFactoryInitialization(beanFactory);
          //refresh做完之后需要做的其他事情。清除上下文资源缓存（如扫描中的ASM元数据），初始化上下文的生命周期处理器，并刷新（找出实现Lifecycle接口的bean并执行start()方法）
            this.finishRefresh();
        } catch (BeansException var9) {
            if (this.logger.isWarnEnabled()) {
                this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
            }

            this.destroyBeans();
            this.cancelRefresh(var9);
            throw var9;
        } finally {
            this.resetCommonCaches();
        }

    }
}
```
