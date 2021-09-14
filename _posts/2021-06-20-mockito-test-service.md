---
title: Mockito 接口测试《第一章》
layout: post
coauthor: 曹德高
tags: [test, mockito]
categories: Unit Test
---

### 什么是Mock测试

Mock测试就是在测试过程中，对于某些不容易构造或者不容易获取的对象，用一个虚拟的对象来创建以便测试的测试方法。什么是不容易构造的对象呢？例如HttpServletRequest，需要在有servlet容器环境中创建获取。那不容易获取的对象呢？如一个JedisCluster，需要准备redis相关环境，然后设置进去等等。

Mock 可以分解在单元测试中耦合的其他类或者接口，它能够帮你模拟这些依赖，并帮你验证所调用的依赖的行为。

关于Mockito的简介之前有同事已经介绍过这里就不在赘述了，大家有兴趣可以自行去官方文档查阅，这里主要带大家了解一些常用的Mock方法。

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>2.23.4</version>
    <scope>test</scope>
</dependency>
为了方便使用Mockito请导入静态类：import static org.mockito.Mockito.*;
并且引入了AssertJ的静态A：import static org.assertj.core.api.Assertions.*;
```

### Service层模拟测试

比如MSF中有一个启动时到com-app获取秘钥信息过来加载类SensitiveDataManager，如果这一步不成功就直接抛异常，直接不让应用启动成功，该类有有个启动时立即调用方法init()用@PostConstruct注解声明，该类还有一个Facade接口是通过Dubbo调用的。如果要完整测试该功能得准备Dubbo环境要ZK，还有第三方接口要先启动且注册到Dubbo ZK中，com-app可能还不属于自己的开发负责范围，怎么启动也不知道，这时候可能自测成本会异常的高，Mock测试就是帮我们解决这一层依赖关系，我们只关系我们要测试的单元模块，我们不需要去准备层层环境才能够测试。

请看下面实例：

```java
import com.google.common.collect.Lists;
import com.qihoo.finance.com.modules.crypto.domain.SensitiveDataSecretKeyDomain;
import com.qihoo.finance.com.modules.crypto.facade.SensitiveDataSecretKeyFacade;
import com.qihoo.finance.msf.api.domain.Response;
import com.qihoo.finance.msf.core.crypto.SensitiveDataManager;

import static org.assertj.core.api.Assertions.*;

import org.junit.Before;
import org.junit.Test;

import static org.mockito.Mockito.*;

import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.powermock.api.mockito.PowerMockito;

import java.lang.reflect.Method;
import java.time.LocalDate;
import java.time.ZoneId;
import java.util.Date;
import java.util.List;

/**
 * SensitiveDataManager功能测试类
 *
 * @author caodegao-jk
 * @date 2020-11-27
 */
public class SensitiveDataManagerTest {

    /**
     * com-app的dubbo接口依赖，我们用挡板代替
     */
    @Mock
    private SensitiveDataSecretKeyFacade sensitiveDataSecretKeyFacade;

    /**
     * 待测试的类
     *
     * @Mock和@InjectMocks的区别，简单的说SensitiveDataSecretKeyFacade是SensitiveDataManager类的属性依赖<br/> SensitiveDataManager需要先注入SensitiveDataSecretKeyFacade才能测试, 所以@Mock标记的类会被注入到@InjectMocks待测试对象中<br/>
     */
    @InjectMocks
    private SensitiveDataManager sensitiveDataManager;

    /**
     * 在测试之前准备一些挡板返回值和参数初始化工作
     */
    @Before
    public void setUp() {
        // mock注解声明及初始化(使用注解必须调用该方法，否则注解只会返回null)
        MockitoAnnotations.initMocks(this);

        // 针对 @Mock 类中的方法进行定制：当调用该接口时返回固定值
        when(sensitiveDataSecretKeyFacade.countAll()).thenReturn(1314L);
        when(sensitiveDataSecretKeyFacade.deleteByPrimaryKey(any(String.class))).thenReturn(1);

        // 如果调用该方法的值是null，则抛异常
        when(sensitiveDataSecretKeyFacade.insertSelective(null)).thenThrow(new RuntimeException("insertSelective prams is null！"));

        // 主要测试com-app中调用的这个方法，先准备查询秘钥的返回初始值挡板。
        when(sensitiveDataSecretKeyFacade.getAllSecretKeys()).thenReturn(convertResponse());

        // SensitiveDataManager的注入方式是构造方法注入，也可以用set注入，但是在SensitiveDataManager中参数标识是final
        // Mockito是不支持final标识的类、参数、方法的，会被直接赋值为null或0
        sensitiveDataManager = new SensitiveDataManager(sensitiveDataSecretKeyFacade);
    }

    private Response<List<SensitiveDataSecretKeyDomain>> convertResponse() {
        Response<List<SensitiveDataSecretKeyDomain>> response = new Response<>();
        List<SensitiveDataSecretKeyDomain> responseList = Lists.newArrayList();
        SensitiveDataSecretKeyDomain domain = new SensitiveDataSecretKeyDomain();
        domain.setAction("action");
        LocalDate dateEffective = LocalDate.of(2018, 9, 24);
        domain.setDateEffective(Date.from(dateEffective.atStartOfDay().atZone(ZoneId.systemDefault()).toInstant()));

        LocalDate dateExpired = LocalDate.of(2021, 9, 24);
        domain.setDateExpired(Date.from(dateExpired.atStartOfDay().atZone(ZoneId.systemDefault()).toInstant()));
        domain.setDeleteFlag("N");
        domain.setId(1L);
        domain.setName("name");
        domain.setSecretKey("secretKey");
        domain.setVersion(1);
        responseList.add(domain);
        response.success(responseList);
        return response;
    }

    @Test
    public void testMock() {
        // 调用前面指定的mock接口
        long result = sensitiveDataSecretKeyFacade.countAll();
        // 检验返回的固定值
        assertThat(1314L).as("校验countAll").isEqualTo(result);

        // 调用前面指定的mock接口
        result = sensitiveDataSecretKeyFacade.deleteByPrimaryKey("aaaa");
        // 检验返回的固定值
        assertThat(1).as("校验deleteByPrimaryKey").isEqualTo(result);

        // 私有方法如果用Mockito是搞不定的，要配合PowerMockito来组合使用
        try {
            //私有方法调用方式一
            SensitiveDataManager spy = PowerMockito.spy(sensitiveDataManager);
            //调用一个不存在的让他抛异常，如果有入参，在方法后面加即可；如果有返回类型可用这个方法返回指定值.thenReturn(返回值);
            PowerMockito.when(spy, "check");
        } catch (Exception e) {
            assertThat(e.fillInStackTrace()).isInstanceOf(org.powermock.reflect.exceptions.MethodNotFoundException.class)
                    .hasMessageContaining("No method found with name");
        }

        try {
            //私有方法调用方式二
            Method method = PowerMockito.method(SensitiveDataManager.class, "init");
            method.invoke(sensitiveDataManager);
        } catch (Exception e) {
            //解密失败
            assertThat(e.fillInStackTrace()).isInstanceOfAny(java.lang.reflect.InvocationTargetException.class, javax.crypto.BadPaddingException.class);
        }

        //判断是否为空
        assertThat(sensitiveDataSecretKeyFacade.insertSelective(new SensitiveDataSecretKeyDomain())).isNotNull();

        //判断是否会抛异常
        assertThatThrownBy(() -> sensitiveDataSecretKeyFacade.insertSelective(null))
                .isInstanceOf(java.lang.RuntimeException.class)
                //异常是否包含该字符串
                .hasMessageContaining("prams is null");

    }
}
```

![image-20201130145237412](/images/mockito-test-service/image-20201130145237412.png)

运行的结果如上图所示

init方法是我没有设置正确的秘钥，所以秘钥解析失败直接保存，打印：这里有错误！insertSelective方法插入为null的参数所有抛了自定义的异常。还有几个调用私有方法的示例，但私有方法本身是有工具方法抛异常的，这里有两种异常判断方式供参考。

我使用了AssertJ工具后把抛异常的代码块都拦截了，这样的实例有益于研发参考且符合单元测试规范。

**为了方便给予示例，我把测试的逻辑都写到了一起，这是不符合规范的，请各位编写单元测试时把各个功能点分开写。**

最后修改为：

```java
    @Test
    public void countAllTest() {
        // 调用前面指定的mock接口
        long result = sensitiveDataSecretKeyFacade.countAll();
        // 检验返回的固定值
        assertThat(1314L).as("校验countAll").isEqualTo(result);

    }

    @Test
    public void deleteByPrimaryKeyTest() {
        // 调用前面指定的mock接口
        int result = sensitiveDataSecretKeyFacade.deleteByPrimaryKey("aaaa");
        // 检验返回的固定值
        assertThat(1).as("校验deleteByPrimaryKey").isEqualTo(result);
    }

    @Test
    public void checkNotThisMethodTest() {
        // 私有方法如果用Mockito是搞不定的，要配合PowerMockito来组合使用
        try {
            //私有方法调用方式一
            SensitiveDataManager spy = PowerMockito.spy(sensitiveDataManager);
            //调用一个不存在的让他抛异常，如果有入参，在方法后面加即可；如果有返回类型可用这个方法返回指定值.thenReturn(返回值);
            PowerMockito.when(spy, "check");
        } catch (Exception e) {
            assertThat(e.fillInStackTrace()).isInstanceOf(org.powermock.reflect.exceptions.MethodNotFoundException.class)
                    .hasMessageContaining("No method found with name");
        }
    }

    @Test
    public void initTest() {
        try {
            //私有方法调用方式二
            Method method = PowerMockito.method(SensitiveDataManager.class, "init");
            method.invoke(sensitiveDataManager);
        } catch (Exception e) {
            //解密失败
            assertThat(e.fillInStackTrace()).isInstanceOfAny(java.lang.reflect.InvocationTargetException.class, javax.crypto.BadPaddingException.class);
        }
    }

    @Test
    public void insertSelectiveParamsNotNullTest() {
        //判断是否为空
        assertThat(sensitiveDataSecretKeyFacade.insertSelective(new SensitiveDataSecretKeyDomain())).isNotNull();
    }

    @Test
    public void insertSelectiveParamsIsNullTest() {
        //判断是否会抛异常
        assertThatThrownBy(() -> sensitiveDataSecretKeyFacade.insertSelective(null))
                .isInstanceOf(java.lang.RuntimeException.class)
                //异常是否包含该字符串
                .hasMessageContaining("prams is null");
    }
```



### 最后

整个Test代码不需要准备Dubbo环境或其他应用依赖，直接等于main方法测试，效率非常快。如果是平时的WEB测试，可能还依赖启动，用junit的Spring测试也会启动应用才能执行到@Test里面的单元测试代码，有些应用很重，启动一次要很久，反复测一个单元测试会让人发疯，导致没人愿意去写单元测试代码。这种Mockito测试方式能让你尽量简化测试代码测试依赖，从而快速验证你关键的方法体。

1. 不建议依赖启动整个项目才能测试的方式。
2. 模拟依赖的接口挡板并初始化好数据即可。
3. 只测试关键的代码功能方法体。
4. 要测试完整流程，先测试完方法体再启动整个应用去整体验证。
5. Mockito是不支持final标识的类、参数、方法的，会被直接赋值为null或0，请注意！！！

本章分享结束。