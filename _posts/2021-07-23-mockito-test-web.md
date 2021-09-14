---
title: Mockito WEB接口测试《第八章》
layout: post
coauthor: 曹德高
tags: [test, mockito]
categories: Unit Test
---

### 前言

前面基本都是接口后端类的测试，这章聊一下web端的http服务如何测试。

我们使用junit测试web的时候不得不起一个完整的服务后调用才能够进入controller代码里面，就如下代码：

```java
@Controller
@RequestMapping
public class WarehouseController {
	@RequestMapping(value = "/web/warehouse/httpDownload")
    public ResponseEntity<String> httpDownload(
    	HttpServletRequest request,
       	HttpServletResponse response,
       	@RequestParam(name = "content", defaultValue = "") String content,
       	@RequestParam(name = "downloadToken", defaultValue = "") String downloadToken,
       	@RequestParam(name = "systemCode", defaultValue = "") String systemCode) {
       	//todo...
    }

    @RequestMapping(value = "/web/warehouse/httpUpload")
    public ResponseEntity<String> httpUpload(
    	HttpServletRequest request,
       	HttpServletResponse response,
        @RequestParam(name = "file") MultipartFile file) {
       	//todo...
    }
}
```

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
//由于是Web项目，Junit需要模拟ServletContext，因此我们需要给我们的测试类加上@WebAppConfiguration。
@WebAppConfiguration
public class WarehouseControllerTest {

    @Autowired
    private WebApplicationContext webApplicationContext;
    private MockMvc mockMvc;

    @Before
    public void setUp() throws Exception {
        //指定WebApplicationContext，将会从该上下文获取相应的控制器并得到相应的MockMvc；
        mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build();
    }
  	
  	@Test
    public void httpDownload() {
        String content = "xxx";
        String systemCode = "xxx";
        String token = "xxx";
        try {
            MvcResult mvcResult = mockMvc.perform(MockMvcRequestBuilders.post("/httpDownload")
                    //contentType参数指定时要注意对应的接受协议类型
                    .contentType(MediaType.APPLICATION_JSON_UTF8)
                    .accept(MediaType.APPLICATION_JSON_UTF8)
                    .param("content", content)
                    .param("systemCode", systemCode)
                    .param("downloadToken", md5))
                    .andExpect(MockMvcResultMatchers.status().isOk())
                    .andDo(MockMvcResultHandlers.print())
                    .andReturn();
            System.out.println(mvcResult.getResponse().getContentAsString());
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}
```

这时候问题就是如果启动整个web应用非常慢，依赖外部存储资源等又非常多，那么该测试会给我们造成非常大的阻碍。

### 分片测试法

所谓分片测试法是指我只让单元测试只模拟启动我所要测试的controller类即可，其他的一概不加载。

```java
import org.apache.commons.io.FileUtils;
import org.junit.Before;
import org.junit.Test;
import org.mockito.MockitoAnnotations;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultHandlers;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
//注解全部都不用
public class WarehouseControllerTest {
    private MockMvc mockMvc;

    @Before
    public void setUp() throws Exception {
        MockitoAnnotations.initMocks(this);
      	//通过MockMvcBuilders.standaloneSetup模拟一个Mvc测试环境，通过build得到一个MockMvc
        mockMvc = MockMvcBuilders.standaloneSetup(new WarehouseController()).build();
    }
  
  	@Test
    public void httpUpload() {
        try {
          	//第一个参数要和Controller里面的@RequestParam MultipartFile指定名称一致
          	MockMultipartFile firstFile = new MockMultipartFile("file", 
                    "filename.txt", "text/plain", "some xml".getBytes());
            MvcResult mvcResult = mockMvc.perform(
              	//调用地址,带文件的方式用multipart
            		MockMvcRequestBuilders.multipart("/web/warehouse/httpUpload")
                            .file(firstFile)
              	//一些传输协议参数
              	//multipart方法里有这个，这里不需要再额外指定.contentType(MediaType.APPLICATION_JSON_UTF8)
              	.accept(MediaType.APPLICATION_JSON_UTF8)
              	//body参数
              	.param("myBody", "xxxx")
              	//头信息
              	.header("myHeader", "{}"))
             //调用接口状态判断，等于一个断言器
             .andExpect(MockMvcResultMatchers.status().isOk())
             .andDo(MockMvcResultHandlers.print())
             .andReturn();
          
          	//有返回值这里获取
            System.out.println(mvcResult.getResponse().getContentAsString());
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```

以上是测试上传和下载文件的方法，这个个例子涵盖了所有的传参和传参类型，是否有返回值等情况，消息体放body还是header例子都包含了。

而且上传文件也不依赖本地的某个文件指定，直接就能模拟一个文件出来。最重要的是脱离了依赖启动整个web容器。
