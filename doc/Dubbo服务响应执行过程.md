## Dubbo服务响应执行过程

### Dubbo总体设计流程

![](img\provider-response.PNG)

### Provider的响应流程：

```java
NettyHandler.messageReceived
  -->AbstractPeer.received
    -->MultiMessageHandler.received
      -->HeartbeatHandler.received
        -->AllChannelHandler.received
          -->ChannelEventRunnable.run //线程池 执行线程
            -->DecodeHandler.received
              -->HeaderExchangeHandler.received
                -->handleRequest(exchangeChannel, request)//网络通信接收处理 
                  -->DubboProtocol.reply
                    -->getInvoker
                      -->exporterMap.get(serviceKey)//从服务暴露里面提取 
                      -->DubboExporter.getInvoker()//最终得到一个invoker
---------------------扩展点--------------
                    -->ProtocolFilterWrapper.invoke
                      -->EchoFilter.invoke
                        -->ClassLoaderFilter.invoke
                          -->GenericFilter.invoke
                            -->TraceFilter.invoke
                              -->MonitorFilter.invoke
                                -->TimeoutFilter.invoke
                                  -->ExceptionFilter.invoke
                                    -->InvokerWrapper.invoke
---------------------扩展点--------------
                                      -->AbstractProxyInvoker.invoke
                                        -->JavassistProxyFactory.AbstractProxyInvoker.doInvoke
                                          --> 进入真正执行的实现类   DemoServiceImpl.sayHello
                                        ....................................
                -->channel.send(response);//把接收处理的结果，发送回去 
                  -->AbstractPeer.send
                    -->NettyChannel.send
                      -->ChannelFuture future = channel.write(message);//数据发回consumer
```

### Consumer接收响应流程：

```java
NettyHandler.messageReceived
  -->AbstractPeer.received
    -->MultiMessageHandler.received
      -->HeartbeatHandler.received
        -->AllChannelHandler.received
          -->ChannelEventRunnable.run //线程池 执行线程
            -->DecodeHandler.received
              -->HeaderExchangeHandler.received
                -->handleResponse(channel, (Response) message);
                  -->HeaderExchangeHandler.handleResponse
                    -->DefaultFuture.received
                      -->DefaultFuture.doReceived
```

分析DefaultFuture.doReceived()方法：

```java
private void doReceived(Response res) {
        lock.lock();
        try {
            response = res;
            if (done != null) {
                done.signal();
            }
        } finally {
            lock.unlock();
        }
        if (callback != null) {
            invokeCallback(callback);
        }
}
```

这里是Dubbo网络通信IO异步转同步的方式。

### Dubbo网络通信IO异步转同步

Dubbo是基于Netty NIO的非阻塞并行调用通信，通信方式有三种类型，参见DubboInvoker.java：

1.异步，有返回值

修改consumer-dubbo.xml

```xml
<dubbo:reference id="demoService" check="false" interface="com.alibaba.dubbo.demo.DemoService">
   <dubbo:method name="sayHello" async="true"/>
</dubbo:reference>
```

修改Consumer调用代码:

```java
DemoService demoService = (DemoService) context.getBean("demoService"); // get remote service proxy
String hello = demoService.sayHello("world"); // call remote method
Future<Object> future = RpcContext.getContext().getFuture();
// get result
System.out.println(future.get());
```

2.异步，无返回值

修改consumer-dubbo.xml

```xml
<dubbo:reference id="demoService" check="false" interface="com.alibaba.dubbo.demo.DemoService">
   <dubbo:method name="sayHello" async="false"/>
</dubbo:reference>
```

修改Consumer调用代码:

```java
DemoService demoService = (DemoService) context.getBean("demoService"); // get remote service proxy
String hello = demoService.sayHello("world"); // call remote method
// get result
System.out.println(hello);
```

3.异步变同步(默认通信方式)

  A.当前线程怎么让它 ”暂停，等结果回来后，再执行”？----Future
  B.socket是一个全双工的通信方式，那么在多线程的情况下，如何知道那个返回结果对应原先那条线程的调用？
  ----通过一个全局唯一的ID来做consumer 和 provider 来回传输。

如DefaultFuture：

```java
// invoke id.
    private final long id;
    private final Channel channel;
    private final Request request;
    private final int timeout;
    private final Lock lock = new ReentrantLock();
    private final Condition done = lock.newCondition();
    private final long start = System.currentTimeMillis();
    private volatile long sent;
    private volatile Response response;
    private volatile ResponseCallback callback;

    public DefaultFuture(Channel channel, Request request, int timeout) {
        this.channel = channel;
        this.request = request;
        this.id = request.getId();
        this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        // put into waiting map.
        FUTURES.put(id, this);
        CHANNELS.put(id, channel);
    }
```

其中的id就是全局通信id，Consumer调用时会设置id，响应时仍然填充返回。
