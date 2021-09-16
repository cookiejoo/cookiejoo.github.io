---
title: Spring @Value注解详解
layout: post
author: 曹德高
tags: [spring]
categories: Spring
---

### 说明

通过@Value注解将配置文件中的属性注入到容器内组件中（可用在@Controller、@Service、@Configuration、@Component等Spring托管的类中）。该注解在Spring容器使用非常广泛，配合Apollo等配置化管理平台更是爽的不得了。

@Value("#{xxxx.key}") 表示支持SpEl表达式，可以调用某个方法来对配置进行增强，甚至可以用其他的@Bean里面的属性给自己。当然还有可以表示常量。

@Value("${xxxx.key}") 表示从配置文件读取值的用法。

@Value("${xxxx.key:val}") 加冒号表示初始值，在没有获取到配置的时候默认用冒号后面这个字符串val当初始值。

### 用法

从Spring properties文件或者配置中心（apollo、nacos等）读取配置。

提示：如果集合类没有默认值话是会报错的，所以建议都给冒号，但空串也会产生一下bug，请使用集合的时候要注意。

```java
//普通字符串获取
@Value("${mysql.user:root}")  
private String mysqlUser;

@Value("${mysql.password:123456}")  
private String mysqlPwd;

@Value("${mysql.pool:8}")  
private Integer pool;

//////////////////////////////////////////////////////////////////////////////
//数组，
@Value("${value.array1:aaa,bbb,ccc}")
private String[] testArray1;
@Value("${value.array2:111,222,333}")
private Integer[] testArray2;
@Value("${value.array3:11.1,22.2,33.3}")
private Double[] testArray3;

//////////////////////////////////////////////////////////////////////////////
//如果要转成List就用El表达式切割法，这种是基本类型的，相信大家也不会让它转自定义对象吧，那这个配置挺奇葩的
@Value("#{'${value.list:ccc,ddd,ggg}'.split(',')}")  
private List<String> valueList;
//也可以用Set
private Set<String> valueSet;
//冒号后面是个空串的话是size会为1，这个判断会容易出bug，可以这么解决空指针问题
@Value("#{'${value.list:}'.empty ? null : '${test.list:}'.split(',')}")  
private List<String> testList;  

//字符串转成Map
//注意：必须有值，空串也会报错，暂时没有找到支持空串的方法，所以使用Map配置值时要考虑这一点
@Value("#{${value.map:{'key1':'value1','key2':'value2','key3':'value3'}}}")
private Map<String, String> valueMap;

//////////////////////////////////////////////////////////////////////////////
//如果有必要将配置转成Bean对象，可以用自定义json解析方法方式来做
package com.cdg.demo;
public class Decoder {  
    public static Map<String, Person> decodeMap(String value) {  
        try {  
            return JSONObject.parseObject(value, new TypeReference<Map<String, Person>>(){});  
        } catch (Exception e) {  
            return null;  
        }  
    }
  
  	public static List<Person> decodeList(String value) {  
        try {  
            return JSONObject.parseArray(value, Person.class);  
        } catch (Exception e) {  
            return null;  
        }  
    }
}  

@Value("#{T(com.cdg.demo.Decoder).decodeMap('${value.person.map:json串}')}")  
private Map<String, Person> map;  
  
@Value("#{T(com.cdg.demo.Decoder).decodeList('${value.person.list:json串}')}")  
private List<Person> list; 

//////////////////////////////////////////////////////////////////////////////
// 其他bean，自定义名称为 myBeans
@Component("myBeans")
@Getter
@Setter
public class OtherBean {
    @Value("cookie")
    private String name;
}
 
// 用法
@Component
@Getter
@Setter
public class MyBean {
  	//如果他不指定myBeans名称，那么就是类名首字母小写：@Value("#{otherBean.name}")
    @Value("#{myBeans.name}")
    private String parentName;
 
}

//////////////////////////////////////////////////////////////////////////////
// 注入文件资源
@Value("classpath:application.properties")
private Resource fileResource;
 
// 注入URL资源
@Value("https://www.caodegao.com")
private Resource urlResource;


//////////////////////////////////////////////////////////////////////////////
//有朋友会用到static的静态变量，静态变量使用要注意，这种是无效的。
@Value("${value.test:}")
public static String test;

//要用set方法来注入
@Value("${value.test:}")
public void setTest(String test) {
    System.out.println("setStr1 ===> " + test);
    MyBean.test = test;
}

```

@Value的常见用法基本上就是示例里面的了，如果还有其他更高级的用法可能官方也会在新版Spring支持推广。
