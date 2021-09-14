---
title: Mockito 异常场景测试技巧《第七章》
layout: post
author: 曹德高
tags: [test, mockito]
categories: Unit Test
---

### 异常场景测试技巧

我们的方法有时候没有异常返回，但是内部的某些方法我们try catch了需要测试覆盖到，这类的场景我们也是非常常见的且着重要注意到的。

#### 模拟执行某方法异常测试

```java
public class TestServiceImpl{
  @Resource
	private MyService myService;
  
	public boolean doSomeThing(){
    boolean isError = false;
    try{
      myService.getName();
    } catch(Exception e) {
      isError = true;
    }
    return isError;
  }
  
  public void throwRuntimeException(){
    try{
      myService.getName();
    } catch (Exception e) {
      throw new RuntimeException("我的异常");
    }
  }
  
  public String getName(){
    myService.getName();
  }
  
  public void openInputStream() throws IOException{
    FileUtils.openInputStream(new File("文件不存在抛异常"));
  }
}

// 单元测试这么做
public class Test{
  @Mock
  MyService myService;
  @InjectMocks
  TestServiceImpl testService;

  @Before
  public void setUp() throws Exception {
    MockitoAnnotations.initMocks(this);
  }
  
  // 令某方法执行异常，但异常被内部补货处理了。
  @Test
  public void doSomeThingMethodReturnTrueTest() {
    doThrow(new RuntimeException()).when(myService).getName();
    boolean result = testService.doSomeThing();
    assertThat(result).as("check result is false!").isEqualTo(false);
  }
  
  // 令某方法执行异常，但是没有被捕获，断言指定的异常类是否包含某关键字
  @Test
  public void throwExceptionTest() {
    doThrow(new RuntimeException()).when(myService).throwException();
    // 使用assertJ的异常断言来测试该方法是否抛了指定异常，还可以判断里面的异常信息是否包含某关键字
    assertThatThrownBy(() -> testService.doSomeThing()).isInstanceOf(RuntimeException.class).hasMessageContaining("我的异常");
  }
  
  // 如上，就判断异常类型
  @Test
  public void getNameThrowExceptionTest() {
    doThrow(new RuntimeException()).when(myService).getName();
    assertThatThrownBy(() -> testService.getName()).isInstanceOf(RuntimeException.class);
  }
  
  // 也可以用catch里面的异常来断言是否是指定的异常
  @Test
  public void throwIOExceptionTest() {
    try{
      testService.openInputStream();
    } catch(Exception e) {
       assertThat(e.fillInStackTrace()).isInstanceOfAny(FileNotFoundException.class);
    }
  }
}
```

建议在很多个方法场景内去编写不同的执行返回或执行异常，setUp只写公共的不会被修改的无状态动作，如果都写在setUp里面会显得臃肿如果要定制某个方法放回不同的数据导致冲突。

异常的处理基本上这几种非常常见常用的，对大家非常有帮助。
