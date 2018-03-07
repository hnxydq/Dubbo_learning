## Dubbo服务引用的设计原理

首先为了分析服务引用的过程， 需要在源码中修改注册中心的地址如下：

dubbo-demo项目的dubbo-demo-consumer/src/test/resources/dubbo.properties

```properties
#dubbo.registry.address=multicast://224.5.6.7:1234
dubbo.registry.address=zookeeper://127.0.0.1:2181
```

dubbo-demo项目的dubbo-demo-consumer/src/main/resources/META-INF/spring/dubbo-demo-consumer.xml

```xml
<!-- 使用multicast广播注册中心暴露发现服务地址 -->
<!--<dubbo:registry address="multicast://224.5.6.7:1234"/>-->
<dubbo:registry protocol="zookeeper" address="127.0.0.1:2181"/>
```

使用两个Idea，分别跑producer和consumer。其中consumer用来断点跟踪。







