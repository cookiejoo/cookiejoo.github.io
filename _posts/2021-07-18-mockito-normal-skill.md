---
title: Mockito 正常场景测试技巧《第六章》
layout: post
author: 曹德高
tags: [test, mockito]
categories: Unit Test
---

### 正常场景测试技巧

在这半个月我写的单元测试用例代码中，遇到了很多问题，比如私有变量、方法内部new对象、静态方法mock等，有些代码我们确实难以模拟的，我把我的解决方式跟大家分享一下。

#### 各种service接口模拟

```java
// Spring里面的接口、Dubbo接口等注入
public class TestServiceImpl{
	@Resource
	private MyService myService;
  
}

// 单元测试这么做
public class Test{
  @Mock
  MyService myService;
  @InjectMocks
  TestServiceImpl testService;

  @Before
  public void setUp() throws Exception {
    // 用@Mock注解即可搞定所有的Spring注入
    MockitoAnnotations.initMocks(this);
    // 再用when().thenReturn()模拟返回值
    // 这类的相对简单
  }
}
```

@Mock为需要注入依赖的类使用，等于在要测试的类里面自动将该类new一个类或接口。

@InjectMocks为待测试的类使用。

#### 各种静态工具类模拟

```java
public class TestServiceImpl{
	/** 
	 * 比如MSF的ServiceContext 或 ApplicationName
	 * 如果不mock出来是为null，又报空指针异常的，Mockito是不支持static、final等修饰符的
	 * 所以要借助org.powermock.api.mockito.PowerMockito 
	 * public static void mockStatic这个工具类配合使用
	 **/
	ApplicationName.getName();
  ServiceContext.getContext().getRequestNo();
  
}

// 单元测试这么做
/**
 * 还需要在头部加注解
 * @PrepareForTest 请注意用了哪些静态类就加进去，最好也要把要测试的这个当前类也加进去(后面讲为什么)
 **/
@RunWith(PowerMockRunner.class)
// 要加这个忽略包，否则启动测试会加载这个虚拟机类导致无法执行单元测试。
@PowerMockIgnore(value = {"javax.management.*"})
@PrepareForTest({ApplicationName.class, ServiceContext.class, TestServiceImpl.class})
public class Test{
  @InjectMocks
  TestServiceImpl testService;

  @Before
  public void setUp() throws Exception {
    MockitoAnnotations.initMocks(this);
    // 这样就保证静态方法不会为null，不会执行测试用例的时候报空指针异常
    PowerMockito.mockStatic(ApplicationName.class);
    PowerMockito.mockStatic(ServiceContext.class);
    when(ApplicationName.getName()).thenReturn("xxx-app");
    when(ServiceContext.getContext()).thenReturn(new ServiceContext());
  }
}
```

#### File文件如何不指定真实文件

我在测试warehouse-app的时候功能就是上传下载，但是我mac book本机有文件想要模拟一个文件，需要指定文件真实路径，但是这个代码提交后，到别人那执行就会报错，因为别人用的是Win，并且别人也没有这个文件，那这个文件就导致了测试依赖第三方资源了。

```java
public class TestServiceImpl{
	public void getFile(String filePath) {
		File file = FileUtils.getFile(firstPath);
   	FileUtils.writeByteArrayToFile(file, "data".getBytes());
  }
}

// 单元测试这么做
@RunWith(PowerMockRunner.class)
// 要加这个忽略包，否则启动测试会加载这个虚拟机类导致无法执行单元测试。
@PowerMockIgnore(value = {"javax.management.*"})
@PrepareForTest({FileUtils.class, TestServiceImpl.class})
public class Test{
  @InjectMocks
  TestServiceImpl testService;

  @Before
  public void setUp() throws Exception {
    MockitoAnnotations.initMocks(this);
    PowerMockito.mockStatic(FileUtils.class);
    // mock一个File类出来，用FileUtils工具的时候直接返回mock的File，这样File就不会为null
    File file = mock(File.class);
    when(FileUtils.getFile(any())).thenReturn(file);
    // FileUtils已经被静态mock了，所以安全的使用里面所有的静态方法
  }
}
```

#### 如何mock在方法里面new的对象

我在测试某些mongodb的工具类的时候，发现有些人写的方法里面new了第三方接口，而且这第三方接口如果手工new还得要很多泛型的参数，困扰了我几个小时，请看我下面的实例和解决方式。

```java
public class TestServiceImpl{
  @Resource(name = "newMongoDataStore")
  private Datastore newMongoDataStore;
  /**
   * 功能是判断fs.files集合中是否存在文件名为{fileId}的文件，没有就将文件写入mongo的fs.files集合中，需要GridFSDBFile和GridFSInputFile类的特殊处理
   **/
	public void sendToMongo(String fileId) {
    EntityDTO entity = new EntityDTO();
    entity.setFileId(fileId);
    newMongoDataStore.save(entity);

    // 按@Mock注解，newMongoDataStore不可能为空，但是返回的db就是空的
    DB db = newMongoDataStore.getDB();
    /**
     * 1：如果db为空，这里new GridFS(db)就抛异常
     * 2：可以用when(newMongoDataStore.getDB())返回一个new DB解决
     * 3：new GridFS里面有一行代码this.filesCollection = db.getCollection(bucketName + ".files");需要一个Collection对象也可以when解决
     * 4：但GridFS gridFS = new GridFS(db);这个是内部new处理的，不属于mock，到下面gridFS.xxx()就开始报空指针了？？？？？
     * 5：请看Test类解决方式
     **/
    GridFS gridFS = new GridFS(db);
    
    GridFSDBFile gridFSDBFile = gridFS.findOne(fileId);
    if (gridFSDBFile == null) {
      GridFSInputFile gridFSInputFile = gridFS.createFile(data);
      gridFSInputFile.put("filename", entity.getFileId());
      //写MongoDB 文件
      gridFSInputFile.save();
    }
  }
}

// 单元测试这么做
@RunWith(PowerMockRunner.class)
@PowerMockIgnore(value = {"javax.management.*"})
// 这里解释一下为什么要加待测试TestServiceImpl的类在这，就是因为方法内部new的mock需要声明在这个注解里面，否则不会生效
@PrepareForTest({FileUtils.class, TestServiceImpl.class})
public class Test{
  @Mock
  DB db;
  @Mock
  DBCollection dbCollection;
  @Mock
  GridFS gridFS;
  @Mock
  GridFSDBFile gridFSDBFile;
  @Mock
  GridFSInputFile gridFSInputFile;
  
  @Mock
  Datastore newMongoDataStore;
  
  @InjectMocks
  TestServiceImpl testService;

  @Before
  public void setUp() throws Exception {
    MockitoAnnotations.initMocks(this);
    // 这两句when解决1、2、3问题
    when(newMongoDataStore.getDB()).thenReturn(db);
    when(db.getCollection(anyString())).thenReturn(dbCollection);
    // 借助PowerMockito解决第4点new GridFS()文问题，等于new GridFS()直接返回@Mock的对象
    PowerMockito.whenNew(GridFS.class).withAnyArguments().thenReturn(gridFS);
    // 经过上面的whenNew声明gridFS就被mock了，下面就能thenReturn其他的类了
    when(gridFS.findOne(anyString())).thenReturn(gridFSDBFile);
    when(gridFS.createFile(any(byte[].class))).thenReturn(gridFSInputFile);
  }
}
```

关键字：PowerMockito.whenNew(GridFS.class).withAnyArguments().thenReturn(gridFS);

#### 如何类里面修改私有属性

根据上面的类我们大概已经知道怎么用方法内部new对象来解决mock问题了，下面mongo还有人另外一种使用方式的，是在Bean初始化的时候再连接mongo，但是我们不想模拟init方法里面的代码，我们能不能也注入一个mock的MongoClient对象呢。。。

```java
public class TestServiceImpl{
  private MongoClient mongoClient;
  private String database;
  @PostConstruct
  private void init() {
    MongoClientFactory mongoClientFactory = new MongoClientFactory(uri, qcmPort);
    this.mongoClient = mongoClientFactory.createMongoClient();
  }

  @PreDestroy
  public void close() {
    if (this.mongoClient != null) {
      this.mongoClient.close();
    }
  }
	public void sendToMongo(String fileId) {
    DB db = this.mongoClient.getDB(this.database);
    GridFS gridFS = new GridFS(db);
  }
}

// 单元测试这么做
@RunWith(PowerMockRunner.class)
@PowerMockIgnore(value = {"javax.management.*"})
@PrepareForTest({FileUtils.class, TestServiceImpl.class})
public class Test{
  @Mock
  DB db;
  @Mock
  DBCollection dbCollection;
  
  @Mock
  Datastore newMongoDataStore;
  
  @InjectMocks
  TestServiceImpl testService;

  @Before
  public void setUp() throws Exception {
    MockitoAnnotations.initMocks(this);
    
    MongoClient mongoClient = mock(MongoClient.class);
    // FieldSetter是利用反射来解决私有属性问题，把Mock出来的MongoClient设置到里面，在@Test测试启动后里面的值也会跟着修改，修改私有属性值有几种方式，我习惯选这种
    FieldSetter.setField(testService, TestServiceImpl.class.getDeclaredField("mongoClient"), mongoClient);
    FieldSetter.setField(testService, TestServiceImpl.class.getDeclaredField("database"), "warehouse");
    
    // 这两句解决new GridFS(db)不为null的问题
    when(newMongoDataStore.getDB()).thenReturn(db);
    when(db.getCollection(anyString())).thenReturn(dbCollection);
  }
}
```

#### 如何测试私有方法

上面测试了修改私有属性，下面来测试私有方法

```java
public class TestServiceImpl{
  
  private void init() {
    System.out.println("success");
  }
}

// 单元测试这么做
public class Test{
  @InjectMocks
  TestServiceImpl testService;

  @Before
  public void setUp() throws Exception {
    MockitoAnnotations.initMocks(this);
  }
  
  @Test
  public void test() throws Exception{
    // 私有方法的执行还是用到PowerMockito工具类，
    Object obj = PowerMockito.method(TestServiceImpl.class, "init").invoke(testService, 如果有参数就附加到这);
  }
}
```

### 总结

上面的实例都是编写单元测试用例常用的方式，就Mockito和PowerMockito结合使用。

注意：mock一个类，但其类中的方法都是返回为空的对象。但可以用when让该方法返回一个自定义mock的对象回来。

