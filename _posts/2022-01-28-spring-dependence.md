---
title: "Spring是如何解决循环依赖"
layout: post
author: 曹德高
tags: [spring]
categories: Spring
---

### **前言**

Spring循环依赖，也就是说类A依赖了类B，类B又依赖类A，那么在项目启动的时候，由于系统不知道先加载A还是B，就会出现循环依赖的错误。

```bash
Error creating bean with name 'AServiceImpl': Bean with name 'AServiceImpl' has been injected into other beans [BServiceImpl,CServiceImpl,EServiceImpl]
```

最好的是重构代码抽取公用部分被大家一起依赖。

或者用@lazy-init属性。在你注入bean时，在互相依赖的两个bean上加上@Lazy注解也可以。

Spring是如何解决循环依赖问题的呢？下面我们从代码的角度的看看它是如何设计的。

### 解析

Spring能够轻松的解决属性的循环依赖正式用到了三级缓存，在AbstractBeanFactory中有注释：

```java
/**
 * Abstract base class for {@link org.springframework.beans.factory.BeanFactory}
 * implementations, providing the full capabilities of the
 * {@link org.springframework.beans.factory.config.ConfigurableBeanFactory} SPI.
 * Does <i>not</i> assume a listable bean factory: can therefore also be used
 * as base class for bean factory implementations which obtain bean definitions
 * from some backend resource (where bean definition access is an expensive operation).
 
 * ## 这里给了链接让我们去看这个DefaultSingletonBeanRegistry类的实现。
 
 * <p>This class provides a singleton cache (through its base class
 * {@link org.springframework.beans.factory.support.DefaultSingletonBeanRegistry},
 * singleton/prototype determination, {@link org.springframework.beans.factory.FactoryBean}
 * handling, aliases, bean definition merging for child bean definitions,
 * and bean destruction ({@link org.springframework.beans.factory.DisposableBean}
 * interface, custom destroy methods). Furthermore, it can manage a bean factory
 * hierarchy (delegating to the parent in case of an unknown bean), through implementing
 * the {@link org.springframework.beans.factory.HierarchicalBeanFactory} interface.
 *
 * <p>The main template methods to be implemented by subclasses are
 * {@link #getBeanDefinition} and {@link #createBean}, retrieving a bean definition
 * for a given bean name and creating a bean instance for a given bean definition,
 * respectively. Default implementations of those operations can be found in
 * {@link DefaultListableBeanFactory} and {@link AbstractAutowireCapableBeanFactory}.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @author Costin Leau
 * @author Chris Beams
 * @since 15 April 2001
 * @see #getBeanDefinition
 * @see #createBean
 * @see AbstractAutowireCapableBeanFactory#createBean
 * @see DefaultListableBeanFactory#getBeanDefinition
 */
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
  ...
}

```

其实我也只能看懂其中一句话并猜测它的意思：是在未知bean的情况下委托给父级。然后通过看源码进行理解，所以就看DefaultSingletonBeanRegistry类是如何实现。

#### 三级缓存

Spring是巧妙的用了三级缓存来存储创建Bean过程中的依赖问题，有点像是发号码牌的意味：卖菜小明的去饭店吃饭，饭店说先点菜点完后给小明一个号码牌坐着等一会叫号，饭店这时发现今天的菜这会11点半了都没准时送达，刚好眼下的小明就是这家饭店进货商，让这小明的赶紧回家拉一车菜来饭店。最后菜也到了，下锅炒了，小明也正常吃到饭了。

我分成四类事件来理解：

- 小明去吃饭
- 点菜
- 号码牌
- 原材料是小明的

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {

	//一级缓存：singletonObjects，存放完全实例化属性赋值完成的Bean，直接可以使用
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	//二级缓存：earlySingletonObjects，存放早期Bean的引用，尚未属性装配的Bean
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
  
	//三级缓存：singletonFactories，三级缓存，存放实例化完成的Bean工厂
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
	...
}
```

*如果要看懂这个示例，可能需要再了解一下Bean的初始化流程是怎么样的，后续我看要不要写一个文章也填补一下。*

1、我们看一下我们获取Bean的方法它做了什么：AbstractAutowireCapableBeanFactory#**doGetBean** 和 DefaultSingletonBeanRegistry#**getSingleton**。

白话一下：对公司而言就只关心获取Bean(程序员)来干活(996)。

```java
  protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  	//一级人间投胎了没有？
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      //没有就去二级地狱生死簿找有没有？
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
          //没有就到三级混沌看有没有精元在重聚
					if (singletonFactory != null) {
            //有就收集三级混沌精元全部上升到二级地狱登记到生死簿中记录原神，清理三级混沌世界存在记录。
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

2、那Bean的创建它做了些什么呢？AbstractAutowireCapableBeanFactory#**doCreateBean** 和 DefaultSingletonBeanRegistry#**addSingletonFactory**。

```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
    ...
    //是否是单例模式(请各位思考一下为啥是单例模式才能用这种3级缓存)，且允许未创建就暴露Bean
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
      //这时候加的是什么 -> 是二级生死簿的引用，就是记个名知道有这么东西
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
    //初始化Bean
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
    ...
  }

--------------------------------------------------------------------------------------------
  protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
      //一级人间投胎完成没有？第一次加载肯定没有
			if (!this.singletonObjects.containsKey(beanName)) {
        //没有放入三级混沌做种子预备队
				this.singletonFactories.put(beanName, singletonFactory);
        //清理干净二级地狱生死簿
				this.earlySingletonObjects.remove(beanName);
        //告知造物主有这么个东西要打算投生程序员并996一生
				this.registeredSingletons.add(beanName);
			}
		}
	}
```

3、Bean初始化缓存升级到一级过程。DefaultSingletonBeanRegistry#**addSingleton**，这里可以理解为给人间种下因果，无论如何都是要投胎的，所以不管是人是妖是灵都被虚拟化了一个位置在人间，等待真正实化的那一刻（变成^(*￣(oo)￣)^）。三个环节一环套一环。

```java
	//当上面的doCreateBean做完后才会到这里把Bean上升到可用状态
  protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
      //升级到人间值得（投胎了）
			this.singletonObjects.put(beanName, singletonObject);
      //混沌和生死簿全部清理干净
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
```

### 总结

**总的过程是先看投胎在人间了没有，没有就通知造物主在三级混沌世界修炼一下，发现三级混沌在苦逼修炼就大发慈悲上升到二级生死簿记载，一切罪孽都消除后就上升一级人间投胎，整个过程需要结合代码来不停的去验证才知道其中奥妙。**

我们搞个例子加图来过一遍吧，加深印象。

假设有A、B、C 三个类依赖，A类有一个属性依赖B，B类有个属性依赖C，C类有个属性依赖A，这个就是循环依赖的示例。

=> 1. A 创建 -> A 构造完成，开始注入属性，发现依赖 B，启动 B 的实例化

=> 2. B 创建 -> B 构造完成，开始注入属性，发现依赖 C，启动 C 的实例化

=> 3. C 创建 -> C 构造完成，开始注入属性，发现依赖 A

![image-20220128175616891](/images/2022-01-28-spring-dependence/image-20220128175616891.png)

- 为什么构造器注入无法解决循环依赖？从中看出来，在Bean调用构造器实例化之前，一二三级缓存并没有Bean的任何相关信息，在实例化之后才放入一级缓存中，因此当getBean的时候缓存并没有命中，这样就抛出了循环依赖的异常了。
- 上面提到多实例Bean为什么无法支持循环依赖(@Scope = prototype)？多实例Bean是每次创建都会调用doGetBean方法，根本没有使用一二三级缓存，肯定不能解决循环依赖。

