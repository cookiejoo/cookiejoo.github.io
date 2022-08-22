---
title: AES PKCS7Padding填充和256位加解密支持
layout: post
author: 曹德高
tags: [java]
categories: Java
---

### 前言

1：我们在java-8中使用AES加解密用PKCS5Padding填充(默认支持)，但是换成PKCS7Padding填充就会报错，原因是jdk-8中没有支持PKCS7Padding的填充。

```
com.qihoo.finance.msf.core.utils.MsfAESUtilTest
============PKCS5Padding===========
java.security.NoSuchAlgorithmException: Cannot find any provider supporting AES/CBC/PKCS7Padding
	at javax.crypto.Cipher.getInstance(Cipher.java:543)
	at com.qihoo.finance.msf.core.utils.MsfAESUtilTest.encryptByCBC(MsfAESUtilTest.java:53)
	at com.qihoo.finance.msf.core.utils.MsfAESUtilTest.main(MsfAESUtilTest.java:26)

Process finished with exit code 0
```

2：现在一般的应用都会使用256位加密key，但是这个在java中有些版本是不支持的，执行加解密会报错。

```java
java.security.InvalidKeyException: Illegal key size or default parameters
```

### 解决方式

**一、AES PKCS7Padding填充加解密支持的二种方式**

1：在jdk安装目录下添加bouncycastle包支持

到该网站下载jar支持包添加到jdk目录`jdk_path/jre/lib/ext`（[ bcprov-jdk18on-171.jar](https://www.bouncycastle.org/download/bcprov-jdk18on-171.jar)）https://www.bouncycastle.org/latest_releases.html#google_vignette

修改/%JAVA_HOME%/jdk/jre/lib/security/java.security这个文件

```bash
security.provider.1=sun.security.provider.Sun
security.provider.2=sun.security.rsa.SunRsaSign
security.provider.3=sun.security.ec.SunEC
security.provider.4=com.sun.net.ssl.internal.ssl.Provider
security.provider.5=com.sun.crypto.provider.SunJCE
security.provider.6=sun.security.jgss.SunProvider
#security.provider.7=com.sun.security.sasl.Provider   #注释这一行
security.provider.7=org.bouncycastle.jce.provider.BouncyCastleProvider #添加这一行
security.provider.8=org.jcp.xml.dsig.internal.dom.XMLDSigRI
security.provider.9=sun.security.smartcardio.SunPCSC
```

2：在AES代码中使用bouncycastle包支持

```java
/**
pom.xml添加依赖
<dependency>
  <groupId>org.bouncycastle</groupId>
  <artifactId>bcprov-jdk18on</artifactId>
  <version>1.71</version>
</dependency>
 */
import org.bouncycastle.jce.provider.BouncyCastleProvider;
public static String encryptByCBC256(String data, String secretKey, String iv) throws Exception {
  ...
  // 在加密方法中添加这一句
  Security.addProvider(new BouncyCastleProvider());
  ...
}
```

示例

```java
package com.qihoo.finance.msf.core.utils;

import org.apache.commons.codec.binary.Base64;
import org.apache.commons.lang3.StringUtils;
import org.bouncycastle.jce.provider.BouncyCastleProvider;

import javax.crypto.Cipher;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.security.Security;


public class MsfAESUtilTest {

    private static final String KEY_ALGORITHM = "AES";
    public static final String SECRET_KEY = "88888888888888888888888888888888";
    public static final String IV = "8cdf051a68d05f24";
    private static final String CBC_CIPHER_ALGORITHM = "AES/CBC/PKCS7Padding";
    private static final String CBC_CIPHER_ALGORITHM_256 = "AES/CBC/PKCS7Padding";
    private static final String CHARSET_NAME = "utf-8";

    public static void main(String[] args) {
        try {

            System.out.println("============PKCS5Padding===========");
            String encrypted = encryptByCBC("caodegao", SECRET_KEY, IV);
            System.out.println("encrypted ： " + encrypted);
            String result = decryptByCBC(encrypted, SECRET_KEY, IV);
            System.out.println("result ： " + result);

            System.out.println("============PKCS7Padding===========");

            encrypted = encryptByCBC256("caodegao", SECRET_KEY, IV);
            System.out.println("encrypted ： " + encrypted);
            result = decryptByCBC256(encrypted, SECRET_KEY, IV);
            System.out.println("result ： " + result);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    public static String encryptByCBC(String data, String secretKey, String iv) throws Exception {
        if (org.apache.commons.lang3.StringUtils.isBlank(data)) {
            return data;
        }
        if (org.apache.commons.lang3.StringUtils.isBlank(iv)) {
            throw new IllegalArgumentException("decryptByCBC iv isBlank");
        }
        byte[] raw = secretKey.getBytes(CHARSET_NAME);
        SecretKeySpec sKeySpec = new SecretKeySpec(raw, KEY_ALGORITHM);
        //算法/模式/补码方式
        Cipher cipher = Cipher.getInstance(CBC_CIPHER_ALGORITHM);
        //使用CBC模式，需要一个向量iv，可增加加密算法的强度
        IvParameterSpec ivParameterSpec = new IvParameterSpec(iv.getBytes());
        cipher.init(Cipher.ENCRYPT_MODE, sKeySpec, ivParameterSpec);
        byte[] encrypted = cipher.doFinal(data.getBytes());

        return Base64.encodeBase64String(encrypted);

    }

    public static String encryptByCBC256(String data, String secretKey, String iv) throws Exception {
        if (org.apache.commons.lang3.StringUtils.isBlank(data)) {
            return data;
        }
        if (org.apache.commons.lang3.StringUtils.isBlank(iv)) {
            throw new IllegalArgumentException("decryptByCBC iv isBlank");
        }
        byte[] raw = secretKey.getBytes(CHARSET_NAME);
        SecretKeySpec keySpec = new SecretKeySpec(raw, "AES");
        Security.addProvider(new BouncyCastleProvider());

        Cipher cipher = Cipher.getInstance(CBC_CIPHER_ALGORITHM_256, "BC");
        cipher.init(Cipher.ENCRYPT_MODE, keySpec, new IvParameterSpec(iv.getBytes()));
        byte[] encrypted = cipher.doFinal(data.getBytes());
        return Base64.encodeBase64String(encrypted);

    }

    public static String decryptByCBC(String data, String secretKey, String iv) throws Exception {
        if (org.apache.commons.lang3.StringUtils.isBlank(data)) {
            return data;
        }
        if (StringUtils.isBlank(iv)) {
            throw new IllegalArgumentException("decryptByCBC iv isBlank");
        }
        byte[] raw = secretKey.getBytes(CHARSET_NAME);
        SecretKeySpec sKeySpec = new SecretKeySpec(raw, KEY_ALGORITHM);
        Cipher cipher = Cipher.getInstance(CBC_CIPHER_ALGORITHM);
        //使用CBC模式，需要一个向量iv，可增加加密算法的强度
        IvParameterSpec ivParameterSpec = new IvParameterSpec(iv.getBytes());
        cipher.init(Cipher.DECRYPT_MODE, sKeySpec, ivParameterSpec);
        byte[] original = cipher.doFinal(Base64.decodeBase64(data));
        return new String(original, CHARSET_NAME);
    }

    public static String decryptByCBC256(String data, String secretKey, String iv) throws Exception {
        if (org.apache.commons.lang3.StringUtils.isBlank(data)) {
            return data;
        }
        if (StringUtils.isBlank(iv)) {
            throw new IllegalArgumentException("decryptByCBC iv isBlank");
        }
        byte[] raw = secretKey.getBytes(CHARSET_NAME);
        SecretKeySpec keySpec = new SecretKeySpec(raw, KEY_ALGORITHM);
        Security.addProvider(new BouncyCastleProvider());
        Cipher cipher = Cipher.getInstance(CBC_CIPHER_ALGORITHM_256, "BC");
        cipher.init(Cipher.DECRYPT_MODE, keySpec, new IvParameterSpec(iv.getBytes()));
        byte[] original = cipher.doFinal(Base64.decodeBase64(data));

        return new String(original, CHARSET_NAME);
    }

}
```

```
运行结果，发现没啥两样

============PKCS5Padding===========
encrypted ： 5yy52rtdRArbCRdA7R0CwQ==
result ： caodegao
============PKCS7Padding===========
encrypted ： 5yy52rtdRArbCRdA7R0CwQ==
result ： caodegao

Process finished with exit code 0
```

**二、解决jdk支持256长度key的两种方式**

1：在jdk安装目录下替换加解密jdk支持

```bash
到Oracle下载https://www.oracle.com/java/technologies/javase-jce8-downloads.html 加密支持包
解压之后得到local_policy.jar & US_export_policy.jar两个jar包，把这两个jar包放到 /%JAVA_HOME%/jdk/lib/security 目录下
jdk

注意：jre\lib\security\policy，在某些版本这个文件夹下能看见两个文件夹，分别是“limited”和“unlimited”，两个文件夹下面的内容都是“local_policy.jar和US_export_policy.jar”这两个东西，我们要用的是“unlimited”下的jar
```


2：升级高版本的jdk（以1.8.x版本为例），但具体的版本我也没有安装过那么多，目前是1.8.151版本不行，换到1.8.202版本后可以。
