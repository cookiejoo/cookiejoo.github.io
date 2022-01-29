---

title: "Spring Boot自动装配流程"
layout: post
author: 曹德高
tags: [spring]
categories: Spring
---

### **前言**

Spring Boot自动装配了解一下，所谓自动装配那自然是要关注@Configuration这个注解啦。

追溯到源码发现有点像SPI技术，了解SPI技术的朋友看起来就没花什么力气了。SPI说白一点就是在某个规定的路径下存放的特定的配置文件（这就是技术规范），里面的内容就是需要应用启动的时候去加载类、初始化类的功能。如果各位朋友看SpringBoot源码到最后，其实也就是在特定的文件路径下特定的文件名，其内容是要初始化的全类路径，就是告诉Spring你要帮我搞定这些自定义类帮我变成你管理的Bean。

这类的实现目的是什么呢？扩展性非常高了，Spring是一个Bean对象管理的高手，不单单是它自己的一些Bean，我们很多类自己实现的，那么你这种第三方的代码也可以给Spring管理，你就按照我Spring的规范去写代码，完了就写规定的配置文件，Spring启动时去整个jar包列表中全部找出这个特定路径文件帮你初始化Bean管理Bean。

*（废话一下：我这人不喜欢用复杂的语言去描述一件事，很多人喜欢用复杂的东西去把一个普通的事情妖魔化，本来就是一加一等于二的事情，硬要套个二次元公式去跟别人讨论。其实你去国外的网站教程人家的解释都是讲说很简单的，很纯粹的思维。）*

### 源码

要了解Spring Boot自动装配，那就要从Spring Boot启动类注解开始，这也可以间接的了解了Spring Boot为啥不需要Tomcat容器去启动(有兴趣自己先去找资料了解一下)。

#### SpringBoot启动类

```java
package com.cdg
@ComponentScan
@SpringBootApplication
public class ServerApplication{
    public static void main(String[] args) {
        SpringApplication.run(ServerApplication.class, args);
    }
}

```

Spring Boot的启动类很简单，就是几个注解，Spring Boot为什么能简化Spring开发，就是因为它觉得单单Spring去开发一个项目pom依赖太多也太庞杂了，它开创了Spring Boot就是简化所有依赖进行整合，所以这个注解就是注解中的注解，也可以说注解功能整合或合并。

ComponentScan也很重要啦，就是哪些类路径是我们这个项目Bean要交给Spring管理的，比如我这里就是com.cdg.\*就配置一下(这里不啰嗦太多了)，一般不写就是ServerApplication下的所有包，比如这个类是在com.cdg下的，那么com.cdg.*所有的包类Spring都扫描去管理。

#### SpringBootApplication注解

其中我们要关注的是SpringBootApplication注解。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {
}
```

#### Spring Boot启动类重要的三个注解

- @SpringBootConfiguration 一看到这个就大概知道是个配置类注解
- @EnableAutoConfiguration 这看英文单词就知道是个开关，就是是否要开始自动装配的功能注解**（我们说自动装配嘛！这个注解意思很明确啦）**
- @ComponentScan 上面有说

#### EnableAutoConfiguration注解

该注解里面@Import(EnableAutoConfigurationImportSelector.class)是关键，是使用EnableAutoConfigurationImportSelector将所有符合条件的@Configuration配置都加载到当前SpringBoot创建并使用的IoC容器。同时借助于Spring框架的一个工具类：SpringFactoriesLoader**（跟java提供的ServiceLoader类似）**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}
```

![image-20220129154109084](/images/2022-01-29-spring-auto-configuration/image-20220129154109084.png)

这个路径就是规范，约定好了这个路径下这个META/spring.factories文件内容就是Spring给用户扩展的配置文件，Spring就去里面帮你加载类且管理起来。我们找个其他jar包来看看基本就知道了。

#### spring.factories文件

如Spring Boot提供了很多数据库集成，比如redis、mysql、mongodb等，我们看看这个spring-boot-autoconfigure包下的配置文件

![image-20220129155243976](/images/2022-01-29-spring-auto-configuration/image-20220129155243976.png)

看看这个写法就可以了

```txt
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jdbc.JdbcRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
```

我们看看Mongodb连接的创建吧，网络很多是Redis，所以多看几个没坏处。

```java
//上面的配置idea里面点进去
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration,\
  
@Configuration
@ConditionalOnClass({ MongoClient.class, com.mongodb.client.MongoClient.class,
		MongoTemplate.class })
@Conditional(AnyMongoClientAvailable.class)
@EnableConfigurationProperties(MongoProperties.class)
@Import(MongoDataConfiguration.class)
@AutoConfigureAfter(MongoAutoConfiguration.class)
public class MongoDataAutoConfiguration {

	private final MongoProperties properties;

	public MongoDataAutoConfiguration(MongoProperties properties) {
		this.properties = properties;
	}

	@Bean
	@ConditionalOnMissingBean(MongoDbFactory.class)
	public MongoDbFactorySupport<?> mongoDbFactory(ObjectProvider<MongoClient> mongo,
			ObjectProvider<com.mongodb.client.MongoClient> mongoClient) {
    ...
	}

	@Bean
	@ConditionalOnMissingBean
	public MongoTemplate mongoTemplate(MongoDbFactory mongoDbFactory,
			MongoConverter converter) {
		return new MongoTemplate(mongoDbFactory, converter);
	}

	@Bean
	@ConditionalOnMissingBean(MongoConverter.class)
	public MappingMongoConverter mappingMongoConverter(MongoDbFactory factory,
			MongoMappingContext context, MongoCustomConversions conversions) {
    ...
	}

	@Bean
	@ConditionalOnMissingBean
	public GridFsTemplate gridFsTemplate(MongoDbFactory mongoDbFactory,
			MongoTemplate mongoTemplate) {
		return new GridFsTemplate(
				new GridFsMongoDbFactory(mongoDbFactory, this.properties),
				mongoTemplate.getConverter());
	}
}
```

MongoProperties这个类不陌生吧，就是在配置文件中找spring.data.mongodb前缀的mongodb连接地址、用户名密码。

```java
@ConfigurationProperties(prefix = "spring.data.mongodb")
public class MongoProperties {
  ...
}
```

总的来说就是创建Mongodb的连接类给你用如：MongoDbFactory、mongoTemplate、gridFsTemplate，**需要额外说明的是它关注的是这个注解@Configuration**。我们要注意的是它有一个@ConditionalOnMissingBean注解，这个注解是如果你已经自己写了这种类，那么Spring就不会创建这个Bean了，会返回你创建的那个Bean，我们一般用Spring的咯。

到了这里大家在写连接Redis、Mongo的例子的时候就会发现，你其实就引入Spring Boot redi/mongo的start依赖，然后配置文件配置一下连接地址等信息，你在Spring中就能使用这些连接模板类了吧，就是由于自动装配的功能帮你把Bean都给创建了，你只管用就行。

### 总结

我以前也是不知道META/spring.factories这个文件的，我也是先去了解java的SPI机制后，发现Spring中有这么个文件，我也不知道这个文件干嘛用的，自从接触Spring Boot自定义start依赖后才去了解这个自动装配。

我是自己自定义配置了一些需要的框架工具方法才用到了这个技术，然后了解这个技术，其实你不用到也不会去了解，也没有机会去了解，没事就去论坛看看、公众号看看一些技术文章或面试题，有助于你去专研底层的原理，为什么呢？因为你也怕假如有一天去面试被问到同样的问题。
