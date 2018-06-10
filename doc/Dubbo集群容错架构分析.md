## Dubbo集群容错架构分析

接下来分析Consumer的请求调用原理。以运行方式启动provider，以debug模式启动consumer。

我们在DemoConsumer类中打断点作为起点来跟踪具体的调用：

```java
public class DemoAction {

    private DemoService demoService;

    public void setDemoService(DemoService demoService) {
        this.demoService = demoService;
    }

    public void start() throws Exception {
        for (int i = 0; i < Integer.MAX_VALUE; i++) {
            try {
                String hello = demoService.sayHello("world" + i);
                System.out.println("[" + new SimpleDateFormat("HH:mm:ss").format(new Date()) + "] " + hello);
            } catch (Exception e) {
                e.printStackTrace();
            }
            Thread.sleep(2000);
        }
    }
}
```

在idea中断点状态下查看表达式的值，可以使用Alt+F8查看。

从demoService.sayHello()说起：

```java
demoService.sayHello("world" + i)
-->InvokerInvocationHandler.invoke
  -->invoker.invoke(new RpcInvocation(method, args))
    -->RpcInvocation//所有请求参数都会转换为RpcInvocation
    -->MockClusterInvoker.invoke(Invocation invocation) //1.进入集群
      -->invoker.invoke(invocation)  //RpcInvocation [methodName=sayHello, parameterTypes=[class java.lang.String], arguments=[world0], attachments={}]
        -->AbstractClusterInvoker.invoke(final Invocation invocation)
          -->list(invocation)
            -->directory.list(invocation)//2.进入目录查找   从this.methodInvokerMap里面查找一个Invoker
              -->AbstractDirectory.list(Invocation invocation)
                -->doList(invocation)
                  -->RegistryDirectory.doList(Invocation invocation)// 从this.methodInvokerMap里面查找一个Invoker
                -->router.route //3.进入路由 
                  -->MockInvokersSelector.route(final List<Invoker<T>> invokers, URL url, final Invocation invocation)
                    -->getNormalInvokers(final List<Invoker<T>> invokers)
          -->ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension("roundrobin")
          -->return doInvoke(invocation, invokers, loadbalance)
            -->FailoverClusterInvoker.doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance)
              -->select(loadbalance, invocation, copyinvokers, invoked) //4.进入负载均衡
                -->AbstractClusterInvoker.select //使用loadbalance选择invoker. a)先lb选择，如果在selected列表中 或者 不可用且做检验时，进入下一步(重选),否则直接返回</br> * b)重选验证规则：selected > available .保证重选出的结果尽量不在select中，并且是可用的
                  -->doselect(loadbalance, invocation, invokers, selected) ////如果只有一个invoker，则直接返回该invoker；如果有两个则退化成轮训；如果更多则做负载均衡如下：
                    -->loadbalance.select
                      -->AbstractLoadBalance.select  //默认做loadbalance
                        -->doSelect
                          -->RoundRobinLoadBalance.doSelect 
                            -->return invokers.get(currentSequence % length)//取模轮循
              -->Result result = invoker.invoke(invocation)
```

集群容错的基本流程：

![cluster_invoker](img/cluster_invoker.png)

### Directory目录服务

包含StaticDirectory和RegistryDirectory。

其中StaticDirectory是静态的，invoker是固定的；RegistryDirectory是注册目录服务，它的Invoker集合来自于ZK，实现了NotifyListener接口，实现了void notify(List<URL> urls);方法，整个过程维护了一个重要变量methodInvokerMap，该map是数据的来源，同时也是notify (RegostryDirectory#notify())的重要操作对象，重点是写操作。（通过doList来完成读操作，通过notify完成写操作, 写操作参照消费者服务引用分析listener.notify(categoryList)）;

下面是读操作：

```java
            -->directory.list//2.进入目录查找   从this.methodInvokerMap里面查找一个Invoker
              -->AbstractDirectory.list
                -->doList(invocation) //完成读操作
                  -->RegistryDirectory.doList// 从this.methodInvokerMap里面查找一个Invoker
                -->router.route //3.进入路由 
                  -->MockInvokersSelector.route
                    -->getNormalInvokers
```

### Router路由规则

### Cluster

FailoverCluster：（默认）失败转移，当出现失败，重试其它服务器，通常用于读操作，但重试会带来更长延迟。

FailfastCluster：快速失败，只发起一次调用，失败立即报错，通常用于非幂等性的写操作。

FailbackCluster：失败自动恢复，后台记录失败请求，定时重发，通常用于消息通知操作。

FailsafeCluster：失败安全，出现异常时，直接忽略，通常用于写入审计日志等操作。

ForkingCluster：并行调用，只要一个成功即返回，通常用于实时性要求较高的操作，但需要浪费更多服务资源。

BroadcastCluster: 广播调用。遍历所有Invokers,逐个调用每个调用catch住异常不影响其他invoker调用

MergeableCluster: 分组聚合，按组合并返回结果，比如菜单服务，接口一样，但有多种实现，用group区分，现在消费方需从每种group中调用一次返回结果，合并结果返回，这样就可以实现聚合菜单项。

AvailableCluster:获取可用的调用。遍历所有Invokers判断Invoker.isAvalible,只要一个有为true直接调用返回，不管成不成功。

### LoadBalance

RandomLoadBalance：随机，按权重设置随机概率。在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

RoundRobinLoadBalance：轮循，按公约后的权重设置轮循比率。存在慢的提供者累积请求问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

LeastActiveLoadBalance：最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差。使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。

ConsistentHashLoadBalance：一致性Hash，相同参数的请求总是发到同一提供者。当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。