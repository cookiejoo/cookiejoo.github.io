---
layout: post
title: netty-http-server
coauthor: 曹德高
tags: [netty]
categories: Netty
---

## 场景

一个应用如果不是web应用，如何使用http接口上传文件下载文件？

<!-- more -->

## 寻找解决方案

我在某应用想开发一个http接口时，发现我的应用不是web应用，想用成熟的组件如spring-web、spring-boot、Tomcat等却望梅止渴，这时候netty就出场了，高性能网络传输框架。

我记得sentinel有类似的接口比如说下发规则到客户端，监听的是8720端口，我去翻了sentinel的源码，确实是用netty做为接口交互的。shardingsphere开源软件也用netty做http和前端交互，我也参考了源码。

![image-20210903153810468](/images/netty-http-server/image-20210903153810468.png)

## 建立服务

```java
 
import com.alibaba.csp.sentinel.concurrent.NamedThreadFactory;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
import io.netty.handler.stream.ChunkedWriteHandler;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Lazy;
import org.springframework.stereotype.Component;
 
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
 
/**
 * @author caodegao
 * @date 2021-04-22
 */
@Slf4j
@Component
@Lazy(false)
public class HttpServer {
 
    @SuppressWarnings("PMD.ThreadPoolCreationRule")
    private final ExecutorService pool = Executors.newSingleThreadExecutor(
            new NamedThreadFactory("netty-command-center-executor"));
 
    //端口自由选择
    @Value("${netty.http.port:8099}")
    private int port;
    //上传文件的大小自由选择：超过700M可能应用也承受不了，异常。。。
    @Value("${netty.http.max.content.length:536870912}")
    private int maxContentLength;
 
    /**
    * @PostConstruct用一个单例默认应用启动的使用就把服务和端口暴露出去。
    */
    @PostConstruct
    public void init() {
        pool.submit(() -> {
            try {
                // 1：bossGroup是门户地址，专门站门口迎宾的
                EventLoopGroup bossGroup = new NioEventLoopGroup(1);
                // 2：workerGroup用于内部编解码、处理业务逻辑的线程组。
                EventLoopGroup workerGroup = new NioEventLoopGroup();
                try {
                    ServerBootstrap bootstrap = new ServerBootstrap();
                    bootstrap.option(ChannelOption.SO_BACKLOG, 1024);
                    bootstrap.group(bossGroup, workerGroup)
                            .channel(NioServerSocketChannel.class)
                            .handler(new LoggingHandler(LogLevel.INFO))
                            .childHandler(new ChannelInitializer<SocketChannel>() {
                                              @Override
                                              protected void initChannel(SocketChannel socketChannel) {
                                                  // 链表模式处理Handler
                                                  ChannelPipeline channelPipeline = socketChannel.pipeline();
                                                	// http编解码处理
                                                  channelPipeline.addLast(new HttpServerCodec());
                                                  //body的内容最大值单位bytes，默认512M
                                                  channelPipeline.addLast(new HttpObjectAggregator(maxContentLength));
                                                  //防止大文件传输java内存溢出，即切割分块传输，默认8K**重要**
                                                  channelPipeline.addLast(new ChunkedWriteHandler());
                                                  //业务类逻辑
                                                  channelPipeline.addLast(new HttpServerHandler());
                                              }
                                          }
                            );
                    Channel channel = bootstrap.bind(port).sync().channel();
                    log.info("App is server on http://127.0.0.1:" + port + '/');
                    channel.closeFuture().sync();
                } finally {
                    bossGroup.shutdownGracefully();
                    workerGroup.shutdownGracefully();
                }
                System.out.println("================================================start");
            } catch (Exception ex) {
                log.error("HttpServer error:", ex);
            }
        });
    }
 
    @PreDestroy
    public void stop() throws Exception {
        //server.close();
        System.out.println("================================================stop");
        pool.shutdownNow();
    }
 
}

```

## 文件上传、下载处理

这个类是处理地址对应的接口方法的，如http://127.0.0.1/httpUpload，给了几种获取参数的方法方式。

```java
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.*;
import io.netty.handler.codec.http.multipart.Attribute;
import io.netty.handler.codec.http.multipart.HttpPostRequestDecoder;
import io.netty.handler.codec.http.multipart.InterfaceHttpData;
import io.netty.util.CharsetUtil;
import lombok.extern.slf4j.Slf4j;
 
import java.io.IOException;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
 
/**
 * Http server handler.
 */
@Slf4j
public final class HttpServerHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
 
    private static final String HTTP_DOWNLOAD = "/httpDownload";
    private static final String HTTP_UPLOAD = "/httpUpload";
 
    /**
     * http入口处理方法
     *
     * @param channelHandlerContext
     * @param request
     */
    @Override
    protected void channelRead0(final ChannelHandlerContext channelHandlerContext, final FullHttpRequest request) {
        String requestPath = request.getUri();
        String requestBody = request.content().toString(CharsetUtil.UTF_8);
        HttpMethod method = request.getMethod();
        log.info("Http request info [uri]:{},[requestBody]:{},[method]{}", requestPath, requestBody, method.name());
        Map<String, String> paramMap = new HashMap<>();
 
        // 下载POST进行获取参数。
        if (HTTP_DOWNLOAD.equalsIgnoreCase(requestPath) && method.equals(HttpMethod.POST)) {
            postParameters(request, paramMap);
            response(channelHandlerContext, paramMap);
            return;
        }
     	 	// 下载GET进行获取参数。
        if (requestPath.contains(HTTP_DOWNLOAD) && method.equals(HttpMethod.GET)) {
            getParameters(requestPath, paramMap);
            response(channelHandlerContext, paramMap);
            return;
        }
        //上传接口只能用POST
        if (requestPath.contains(HTTP_UPLOAD) && method.equals(HttpMethod.POST)) {
            // 上传逻辑另外给
            upload(channelHandlerContext, request);
            return;
        }
        response("Not support request!".getBytes(),
                channelHandlerContext, HttpResponseStatus.BAD_REQUEST);
    }
 
    /**
     * post获取参数
     *
     * @param request
     * @param paramMap
     */
    private void postParameters(FullHttpRequest request, Map<String, String> paramMap) {
        HttpPostRequestDecoder decoder = new HttpPostRequestDecoder(request);
        decoder.offer(request);
        List<InterfaceHttpData> paramList = decoder.getBodyHttpDatas();
        try {
            for (InterfaceHttpData param : paramList) {
                Attribute data = (Attribute) param;
                paramMap.put(data.getName(), data.getValue());
            }
        } catch (IOException e) {
            log.error("postParameters Error:", e);
        }
    }
 
    /**
     * get获取参数
     *
     * @param requestPath
     * @param paramMap
     */
    private void getParameters(String requestPath, Map<String, String> paramMap) {
        // 是GET请求
        QueryStringDecoder decoder = new QueryStringDecoder(requestPath);
        decoder.parameters().entrySet().forEach(entry -> {
            // entry.getValue()是一个List, 只取第一个元素
            paramMap.put(entry.getKey(), entry.getValue().get(0));
        });
    }
 
    
    public void response(byte[] bytes, final ChannelHandlerContext ctx, final HttpResponseStatus status) {
        byte[] content = bytes;
        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, status, Unpooled.copiedBuffer(content));
        response.headers().set("content-type", "text/plain;charset=UTF-8");
        setContentLength(response, response.content().readableBytes());
        response.headers().set("connection", "keep-alive");
        //写完刷新流
        ctx.writeAndFlush(response);
    }
 
 
    /**
     * 重新http异常处理
     *
     * @param ctx
     * @param cause
     */
    @Override
    public void exceptionCaught(final ChannelHandlerContext ctx, final Throwable cause) {
        if (cause.getMessage().equalsIgnoreCase("Connection reset by peer")) {
            log.warn("Http request handle occur error localAddress={} remoteAddress={}: Connection reset by peer", ctx.channel().localAddress(), ctx.channel().remoteAddress());
        } else {
            log.warn("Http request handle occur error localAddress={} remoteAddress={}:", ctx.channel().localAddress(), ctx.channel().remoteAddress(), cause);
        }
        ResponseUtil.response(cause.toString(), ctx, HttpResponseStatus.INTERNAL_SERVER_ERROR);
        ctx.close();
    }
}
```

## 文件上传解析工具

```java
//FileBody是用于存储上传文件是多个文件的情况下，所以文件是List，参数解析后用map存储下来，给业务逻辑用。
@Getter
@Setter
public class FileBody {
    List<FileUpload> fileUploadList;
    Map<String, String> paramMap;
}
 
 
import io.netty.channel.*;
import io.netty.handler.codec.http.*;
import io.netty.handler.codec.http.multipart.*;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.io.FileUtils;
import org.apache.commons.io.FilenameUtils;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
 
import javax.annotation.Resource;
import java.io.*;
import java.util.Map;
@Slf4j
public class UploadUtil{
    public FileBody getFileUpload(FullHttpRequest request) throws IOException {
        //创建HTTP对象工厂
        HttpDataFactory factory = new DefaultHttpDataFactory(true);
        //使用HTTP POST解码器
        HttpPostRequestDecoder httpDecoder = new HttpPostRequestDecoder(factory, request);
        httpDecoder.setDiscardThreshold(0);
        //获取HTTP请求对象
        //加载对象到加吗器。
        httpDecoder.offer(request);
        //存放文件对象
        FileBody fileBody = new FileBody();
        
        //通过迭代器获取HTTP的内容
        java.util.List<InterfaceHttpData> InterfaceHttpDataList = httpDecoder.getBodyHttpDatas();
        for (InterfaceHttpData data : InterfaceHttpDataList) {
          //如果数据类型为文件类型，则保存到fileUploads对象中
          if (data != null && InterfaceHttpData.HttpDataType.FileUpload.equals(data.getHttpDataType())) {
            FileUpload fileUpload = (FileUpload) data;
            fileBody.getFileUploadList().add(fileUpload);
          }
          //如果数据类型为参数类型，则保存到body对象中
          if (data.getHttpDataType() == InterfaceHttpData.HttpDataType.Attribute) {
            Attribute attribute = (Attribute) data;
            fileBody.getParamMap().put(attribute.getName(), attribute.getValue());
          }
        }
        return fileBody;
    }
 
 
    public void upload(ChannelHandlerContext ctx, FullHttpRequest request) {
        try {
            FileBody fileBody = getFileUpload(request);
            for (FileUpload file : fileBody.getFileUploadList()) {
                //file这里就是真实文件了，自由处理业务逻辑
            }
            response("返回上传的情况".getBytes(), ctx, HttpResponseStatus.OK);
        } catch (Exception e){
            
        }
    }
}
```

