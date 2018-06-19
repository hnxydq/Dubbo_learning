## Dubbo如何实现SOA的服务降级

### 服务开关

对于在线商城，在下单交易环节，可能需要调用A、B、C三个接口来完成，其中A和B是必须的，C只是附加功能（如在下单的时候推荐相关商品或push消息），可有可无，在平时系统没有压力，容量充足的情况下，调用下没问题，但是在类似618店庆之类的大促环节，系统满负荷运行，此时完全可以不调用C接口，怎么实现这个功能？ 改代码？---加个服务开关

### 服务降级

使用dubbo在进行服务调用时，可能由于各种原因（服务器宕机/网络超时/并发数太高等），调用中就会出现RpcException，调用失败。

服务降级就是当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务和页面有策略的降级，以此释放服务器资源以保证核心任务的正常运行。换句话说，就是指在由于非业务异常导致的服务不可用时，可以返回默认值，避免异常影响主业务的处理。

1. 屏蔽：在大促，促销活动的可预知情况下，例如双11活动。采用直接屏蔽接口访问。（可知）

​        mock=force:returnnull

   2.容错：当系统出现非业务异常(比如并发数太高导致超时，网络异常等)时，不对该接口进行处理。（不可知）

​      mock=fail:returnnull

### Dubbo服务降级配置

####mock配置方式：

dubbo官方文档上使用一个mock配置，实现服务降级。mock只在出现非业务异常(比如超时，网络异常等)时执行。mock的配置支持两种，一种为boolean值，默认的为false。如果配置为true，则缺省使用mock类名，即类名+Mock后缀；另外一种则是配置”return null”，可以很简单的忽略掉异常。

mock配置在调用方，服务降级不需要对服务方配置产生修改。下面举个例子说明，mock的配置：

```java
/**接口定义*/
public interface FooService {
    public String doSomething(String str);
}

/**实现类*/
public class FooServiceImpl implements FooService {
    public String doSomething(String str) {
        System.out.println("service invoke: doSomething ," + str);
        return "service invoke: doSomething";
    }
}
```

provider.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans.xsd        http://code.alibabatech.com/schema/dubbo        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="hello-world-app" />
    <!-- 使用multicast广播注册中心暴露服务地址 -->
    <dubbo:registry address="zookeeper://127.0.0.1:2181" />
    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />
    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="com.test.service.FooService" ref="fooServerImpl" timeout="10000" />
    <!-- 和本地bean一样实现服务 -->
    <bean id="fooServerImpl" class="com.test.serviceimpl.FooServerImpl" />
</beans>
```

consumer.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans.xsd        http://code.alibabatech.com/schema/dubbo        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
    <dubbo:application name="dubbo-consumer"  />
    <dubbo:registry address="zookeeper://127.0.0.1:2181" />
    <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
    <dubbo:reference id="fooService" interface="com.test.service.FooService"  timeout="10000" check="false" mock="return null">
    </dubbo:reference>
</beans>
```

测试在调用端调用服务两个方法，当服务端正常启动时，程序获得正常返回值；当服务提供方没有启动（模拟服务不可用状态）,调用方依然正常运行，调用doSomething获取返回值时null。

####mock实现接口方式

上面在`<dubbuo:reference>` 中配置`mock="retrun null"` 的配置，在服务降级时会对service中的所有方法做统一处理，即都返回null。但是有的时候我们需要一些方法在服务不可用时告诉我们一些其他信息，以便做其他处理。如更新/删除等。要有较好的区分，可以通过以下的方式。

配置`mock="true"` ,同时实现mock接口，类名要注意命名规范：接口名+Mock后缀。此时如果调用失败会调用Mock实现。mock实现需要保证有无参的构造方法。

配置mock=”true”的情况，对于上面的例子即在FooService的同个路径下，添加类FooServiceMock，实现如下：

```java
public class FooServiceMock implements FooService {
    public String doSomething(String str) {
        return null;
    }
}
```

#### Dubbo服务降级具体实现

通过Dubbo的Filter对Dubbo进行扩展，从而使得每次服务发起调用都可以得到监控，从而可以监控每次服务的调用。

<u>对自动判断服务提供端是否宕机：通过一个记录器对每个方法出现RPC异常进行记录，并且可以配置在某个时间段内连续出现都少个异常可判定为服务提供端出现了宕机，从而进行服务降级。</u>

<u>自动恢复远程服务调用：通过配置检查服务的频率来达到定时检查远程服务是否可用，从而去除服务降级。</u>

####判断降级相关配置

降级配置分配为应用级别，接口级别，方法级别 。dubbo相关参数配置在dubbo.properties中,默认是在classpath根目录，也可以通过-Ddubbo.properties.file来指定该文件路径。

1.应用级别

```properties
dubbo.reference.default.break.limit:该参数是配置一个方法在指定时间内出现多少个异常则判断为服务提供方宕机 
dubbo.reference.default.retry.frequency:该参数配置重试频率，比如配置100，则表示没出现一百次异常则尝试一下远程服务是否可用 
dubbo.reference.circuit.break:服务降级功能开关，默认是false，表示关闭状态，可以配置为true
```

2.接口级别

```properties
dubbo.reference.fullinterfacename.break.limit:同上面dubbo.reference.default−break−limit，指定某个接口dubbo.reference.{fullinterfacename}.retry.frequency:同上面 
dubbo.reference.${fullinterfacename}.circuit.break:服务降级功能开关，默认是false，表示关闭状态，可以配置为true
```

3.方法级别

```properties
dubbo.reference.fullinterfacename.{methodName}.break.limit:同上面dubbo.reference.default-break-limit，指定某个接口的某个方法 
dubbo.reference.fullinterfacename.{methodName}.retry.frequency:同上面dubbo.reference.default-retry-frequency，指定某个接口的某个方法 
dubbo.reference.fullinterfacename.{methodName}.circuit.break:服务降级功能开关，默认是false，表示关闭状态，可以配置为true
```

