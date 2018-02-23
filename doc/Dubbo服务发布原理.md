## Dubbo服务发布原理

观察服务启动日志：

![demo-provider](img/demo-provider.PNG)

从Provider启动日志可以看到，主要做了5个发布动作：

1.暴露本地服务

2.暴露远程服务

3.启动Netty

4.打开连接Zookeeper

5.到zookeeper注册

6.监听zookeeper







