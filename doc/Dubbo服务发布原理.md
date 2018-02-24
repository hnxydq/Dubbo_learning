## Dubbo服务发布原理

修改dubbo-demo子项目的dubbo-demo-provider模块的dubbo-demo-provider.xml中注册中心的配置：

```xml
<!-- 使用multicast广播注册中心暴露服务地址 -->
<!--<dubbo:registry address="multicast://224.5.6.7:1234"/>-->
<dubbo:registry protocol="zookeeper" address="127.0.0.1:2181"/>
```

同时修改dubbo-config子项目的dubbo-config-spring模块的test/resources/dubbo.properties文件中：

```properties
#dubbo.registry.address=multicast://224.5.6.7:1234
dubbo.registry.address=zookeeper://127.0.0.1:2181
```

需要外部下载启动zookeeper组件。

然后启动dubbo-demo-provider/test/java下的DemoProvider观察服务启动日志：

![demo-provider](img/demo-provider.PNG)

从Provider启动日志可以看到，主要做了6个发布动作：

```
1.暴露本地服务
2.暴露远程服务
3.启动Netty
4.打开连接Zookeeper
5.到zookeeper注册
6.监听zookeeper
```

### 暴露本地服务

暴露的服务，其实就是dubbo-demo-provider.xml中配置的service：

```xml
<!-- 和本地bean一样实现服务 -->
<dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>
```

dubbo:service是在dubbo-config-spring下resources/META-INF下的dubbo.xsd约束schema文件中定义的。

而处理Handler在dubbo-config-spring下的schema包下的DubboNamespaceHandler.java中处理：

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
    }
}
```

可以看到，service标签对应的Bean为ServiceBean。看看ServiceBean类继承关系：

```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener, BeanNameAware 
```

这里ServiceBean继承自ServiceConfig，并且实现了Spring框架的ApplicationListener接口。

```java
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    void onApplicationEvent(E var1);
}
```

所以在Spring框架启动时，会去回调执行ServiceBean的onApplicationEvent(e)方法。

```java
public void onApplicationEvent(ApplicationEvent event) {
        if (ContextRefreshedEvent.class.getName().equals(event.getClass().getName())) {
            if (isDelay() && !isExported() && !isUnexported()) {
                if (logger.isInfoEnabled()) {
                    logger.info("The service ready on spring started. service: " + getInterface());
                }
                export();
            }
        }
}
```













