---
layout:     post
title:      "Californium开源框架之源码分析（二）"
subtitle:   "coap模块，协议中的常量定义与消息模型。"
date:       2017-06-01 22:00:00
author:     "Wudashan"
header-img: "img/post-bg-californium-framework-analysis.jpg"
catalog: true
tags:
    - Californium
    - 开源框架
    - 源码分析
    - CoAP
---

> 项目源码地址：[https://github.com/eclipse/californium](https://github.com/eclipse/californium)

# coap包

coap包目录下，主要是CoAP协议中定义的常量和消息基本模型。

![](http://o7x0ygc3f.bkt.clouddn.com/Californium%E5%BC%80%E6%BA%90%E6%A1%86%E6%9E%B6%E5%88%86%E6%9E%90/coap%E5%8C%85%E7%B1%BB%E5%9B%BE.png)

## CoAP类

该类定义了CoAP报文的一些常量，以及按协议要求定义了报文中各字段的格式。包括：

 - 消息类型：CON，NON，ACK，RST
 - 请求码：GET，POST，PUT，DELETE
 - 响应码：成功2.XX，客户端错误4.XX，服务端错误5.XX

## OptionNumberRegistry类

根据RFC 7252的第12.2章节，该类定义了CoAP报文中Option字段支持的值。其值如下表格：

```
+--------+------------------+-----------+ 
| Number | Name             | Reference | 
+--------+------------------+-----------+ 
|      0 | (Reserved)       | [RFC7252] | 
|      1 | If-Match         | [RFC7252] |
|      3 | Uri-Host         | [RFC7252] | 
|      4 | ETag             | [RFC7252] |  
|      5 | If-None-Match    | [RFC7252] |  
|      6 | Observe          |           |  
|      7 | Uri-Port         | [RFC7252] |  
|      8 | Location-Path    | [RFC7252] | 
|     11 | Uri-Path         | [RFC7252] |
|     12 | Content-Format   | [RFC7252] |
|     14 | Max-Age          | [RFC7252] | 
|     15 | Uri-Query        | [RFC7252] |  
|     17 | Accept           | [RFC7252] |   
|     20 | Location-Query   | [RFC7252] |  
|     23 | Block2           |           |    
|     27 | Block1           |           | 
|     28 | Size2            |           | 
|     35 | Proxy-Uri        | [RFC7252] |  
|     39 | Proxy-Scheme     | [RFC7252] | 
|     60 | Size1            | [RFC7252] |
|    128 | (Reserved)       | [RFC7252] | 
|    132 | (Reserved)       | [RFC7252] |  
|    136 | (Reserved)       | [RFC7252] |   
|    140 | (Reserved)       | [RFC7252] |    
+--------+------------------+-----------+ 
```

## MediaTypeRegistry类

根据RFC 7252的第12.3章节，该类定义了CoAP报文中Option字段的Content-Format选项支持的值。其值如下表格：

```
+--------------------------+----------+----+----------------------+
| Media type               | Encoding | ID | Reference            |
+--------------------------+----------+----+----------------------+
| text/plain;              | -        |  0 | [RFC2046] [RFC3676]  |
| charset=utf-8            |          |    | [RFC5147]            |
| application/link-format  | -        | 40 | [RFC6690]            |
| application/xml          | -        | 41 | [RFC3023]            |
| application/octet-stream | -        | 42 | [RFC2045] [RFC2046]  |
| application/exi          | -        | 47 | [REC-exi-20140211]   |
| application/json         | -        | 50 | [RFC7159]            |
+--------------------------+----------+----+----------------------+
```

## MessageFormatException类

该类继承RuntimeException类，即属于运行时（非受检）异常。当解析二进制形式的CoAP报文失败时，程序就会抛出该异常。


## BlockOption类

该类表示的是CoAP报文中Option字段的Block1和Block2值，由于这两个选项比较特殊，所以单独封装成一个类。


## Option类

该类是个Option字段的通用表示类。请求报文和响应报文都有可能携带多个Option，可以通过Option的序号判断该Option是必须还是可选，安全还是非安全。如果某个Option选项是安全的，那么还可以判断它是否是可缓存的。算法如下：

```
Critical = (OptionNumber & 1);
UnSafe = (OptionNumber & 2);
NoCacheKey = ((OptionNumber & 0x1e) == 0x1c)
```

说起来非常抽象，让我们直接看一个实例，Option序号二进制表示形式如下：

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
|           | NoCacheKey| U | C |
+---+---+---+---+---+---+---+---+
```

从OptionNumberRegistry类查询到Uri-Host的序号为3，转换为二进制就是`000 000 1 1`。

 - Critical = (00000011 & 00000001) = 1，所以Uri-Host是必选的；
 - UnSafe = (00000011 & 00000010) != 0，所以Uri-Host是不安全的；
 - NoCacheKey = (00000011 & 00011110) != 00011100，所以Uri-Host不是非缓存键。

## OptionSet类

该类是一个POJO类，其含义为CoAP协议里消息体的Option集合，我们从它的成员变量就可以看出来：

```
public class OptionSet {
    private List<byte[]> if_match_list;
    private String       uri_host;
    private List<byte[]> etag_list;
    private boolean      if_none_match; // true if option is set
    private Integer      uri_port; // null if no port is explicitly defined
    private List<String> location_path_list;
    private List<String> uri_path_list;
    private Integer      content_format;
    private Long         max_age; // (0-4 bytes)
    private List<String> uri_query_list;
    private Integer      accept;
    private List<String> location_query_list;
    private String       proxy_uri;
    private String       proxy_scheme;
    private BlockOption  block1;
    private BlockOption  block2;
    private Integer      size1;
    private Integer      size2;
    private Integer      observe;
    private List<Option> others; // Arbitrary options
    ...
}
```

它涵盖了RFC 7252中定义的所有Option字段，包括上面提到的通用Option类和单独BlockOption类，也都统一的封装在该类中。

## Message类

终于到我们coap包中最重要的一个类了，该类是所有CoAP消息的基类。CoAP消息一共有三个子类：请求消息Request类、响应消息Response类、空消息EmptyMessage类。每个CoAP消息都包括消息类型Type、MID、token、OptionSet和payload等等。

除此之外，一个消息可以被应答、拒绝、取消、超时，所以在该类也定义了相应的成员变量来表示：

```
public abstract class Message {
    // 表示消息是否已应答
    private boolean acknowledged;

    // 表示消息是否被拒绝
    private boolean rejected;

    // 表示消息是否被取消
    private boolean canceled;

    // 表示消息是否已超时
    private boolean timedOut; // Important for CONs
    ...
}
```

该类使用了观察者设计模式。通过`Message.addMessageObserver()`加入观察者，当消息状态（应答、拒绝、取消、超时）出现变更时，会通知相应的MessageObserver类。从具体的内部实现代码即可发现：

```
public void setCanceled(boolean canceled) {
    this.canceled = canceled;
    if (canceled) {
        // 逐个通知观察者
        for (MessageObserver handler : getMessageObservers()) {
            // 回调观察者的相应方法
            handler.onCancel();
        }
    }
}
```

## Request类

该类继承自Message类，表示一个CoAP请求消息。其消息类型Type为CON或者NON，请求码Code为GET、PUT、POST、DELETE的其中一种。一个请求消息需要通过Endpoint类来发送到它的目的地去。若不指定Endpoint，则框架会通过EndpointManager来生成一个默认的Endpoint。通常由服务端回复一个Response类，即一个对应的CoAP响应消息。客户端可以发起一个同步请求：

```
// 创建一个GET请求并发送
Request request = new Request(Code.GET);
request.setURI("coap://example.com:5683/sensors/temperature");
request.send();
// 同步等待服务器响应
Response response = request.waitForResponse();
```

当然，有同步请求就有异步请求，通过添加观察者到Request类中，当请求消息收到响应时回调观察者：

```
// 创建一个GET请求
Request request = new Request(Code.GET);
request.setURI("coap://example.com:5683/sensors/temperature");
// 添加观察者
request.addMessageObserver(new MessageObserverAdapter() {
    public void responded(Response response) {
        // 收到响应时回调该方法
        if (response.getCode() == ResponseCode.CONTENT) {
            System.out.println("Received :" + response.getPayloadString());
        } else {
            // 错误处理
        }
    }
});
// 发送请求
request.send();
```

我们还可以自定义Request类中Option的值，例如：

```
// 创建一个POST请求
Request post = new Request(Code.POST);
post.setPayload("Plain text");
// 自定义Option的值
post.getOptions()
    .setContentFormat(MediaTypeRegistry.TEXT_PLAIN)
    .setAccept(MediaTypeRegistry.TEXT_PLAIN)
    .setIfNoneMatch(true);
// 发起同步请求
String response = post.send().waitForResponse().getPayloadString();
```

## Response类

该类继承自Message类，表示一个CoAP响应消息。一个响应消息可以是ACK、CON或NON三种类型。响应体里包含着响应码，这与HTTP响应码类似。该类还提供了一个静态工厂方法来生成Response类：

```
public static Response createResponse(Request request, ResponseCode code) {
    Response response = new Response(code);
    response.setDestination(request.getSource());
    response.setDestinationPort(request.getSourcePort());
    return response;
}
```

从静态工厂方法我们看到，只设置了响应消息的响应码、目的地地址和端口号，而Type、MID和token都没有进行赋值。这是因为，Type和MID通常会在`ReliabilityLayer`类里自动设置，token则通常在`Matcher`类里自动设置。

## EmptyMessage类

该类继承自Message类，表示一个CoAP空消息。一个空消息只能是ACK或者RST两种类型。所以该类也只提供了两个静态工厂方法：`EmptyMessage.newACK()`和`EmptyMessage.newRST()`。

## MessageObserver接口

该接口是一个观察者，当事件发生变化时调用相应的方法：

 - onResponse() 当收到响应时回调
 - onAcknowledgement() 当收到应答时回调
 - onReject() 当请求消息被拒绝时回调
 - onTimeout() 当客户端停止重传且仍然没有从对端收到任何消息时回调
 - onCancel() 当消息被取消时回调
 
正如我们前面在Message类所提到的，可以通过`Message.addMessageObserver()`方法来注册观察者。需要注意的是，CoAP协议支持客户端对服务端的资源Resource进行订阅（当资源发生变化时服务端主动发送响应消息给客户端），这种订阅是通过`NotificationListener`接口进行监听的，大家不要和这个类搞混了。

## MessageObserverAdapter类

该类是一个抽象类，实现了MessageObserver接口。看这个类的名字，还以为使用到了适配器设计模式，仔细看源代码才发现，该类只是将MessageObserver接口中的所有方法都提供空实现：

```
public abstract class MessageObserverAdapter implements MessageObserver {
    @Override
    public void onResponse(Response response) {
        // 什么也不做
    }
    @Override
    public void onTimeout() {
        // 什么也不做
    }
    ...
}
```

大家可能会比较疑惑框架为什么要提供一个这样看上去无意义的抽象类，其实是因为如果开发者在编写自己的MessageObserver实现类时，可能只关注onResponse()和onTimeout()方法，那么为了减少其他不必要的代码，可以直接继承MessageObserverAdapter抽象类，然后按需覆盖这两个方法。

---

# 系列文章

[Californium开源框架之源码分析（一）—— 整体认识](http://wudashan.cn/2017/05/21/Californium-Framework-Analysis-01/) 

[Californium开源框架之源码分析（二）—— coap包](http://wudashan.cn/2017/06/01/Californium-Framework-Analysis-02/) <-- 当前位置

[Californium开源框架之源码分析（三）—— observe包](http://wudashan.cn/2017/06/05/Californium-Framework-Analysis-03/)

[Californium开源框架之源码分析（四）—— server包](http://wudashan.cn/2017/06/16/Californium-Framework-Analysis-04/)

[Californium开源框架之源码分析（五）—— network包（上）](http://wudashan.cn/2017/07/02/Californium-Framework-Analysis-05/)

[Californium开源框架之源码分析（六）—— network包（下）](http://wudashan.cn/2017/07/07/Californium-Framework-Analysis-06/)

[Californium开源框架之源码分析（七）—— core包](http://wudashan.cn/2017/07/09/Californium-Framework-Analysis-07/)

[Californium开源框架之源码分析（八）—— element包](http://wudashan.cn/2017/07/12/Californium-Framework-Analysis-08/)
