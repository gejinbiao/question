# dubbo的实现原理

# dubbo的介绍

dubbo是阿里巴巴公司开源的一个高性能优秀的服务框架，使得应用可通过高性能的RPC实现服务的输出和输入功能，可以和Spring框架无缝集成。

dubbo框架是基于Spring容器运行的。

# RPC远程过程调用

远程过程调用协议是一种通过网络从远程计算机程序上请求服务，而不需要了解网络底层技术的协议。

RPC协议假定某些传输协议的存在，如TCP或者UDP，为通信程序之间携带信息数据。

在OSI网络通信模型中，RPC跨越了传输层和应用层。

RPC的优点：使得开发包括网络分布式多程序在内的应用程序更加容易。

# dubbo角色介

![NXcxwd.png](https://s1.ax1x.com/2020/07/03/NXcxwd.png)

**注册中心（registry）**：生产者在此注册并发布内容，消费者在此订阅并接收发布的内容。

**消费者（consumer）**：客户端，从注册中心获取到方法，可以调用生产者中的方法。

**生产者（provider）**：服务端，生产内容，生产前需要依赖容器（先启动容器）。

**容器（container）**：生产者在启动执行的时候，必须依赖容器才能正常启动（默认依赖的是spring容器），dubbo技术不能脱离spring框架。2.5.3版本的dubbo默认依赖spring2.5版本，可以选用spring4.5以下的版本2.5.7版本的dubbo默认依赖spring4.3.10版本，可以选择任意的spring版本。

**监控中心（monitor）**：是dubbo提供的一个jar工程。主要功能是监控服务端和消费端的使用数据。
如：服务端是什么，有多少接口，多少方法，调用次数，压力信息等，客户端有多少，调用过哪些服务端，调用了多少次等。

# dubbo执行流程（以上图为例）

0.start：启动Spring容器时，自动启动dubbo的Provider

1.register：dubbo的Provider在启动后自动会去注册内容。注册的内容包括：

　　　　　　1.1 Provider的IP

　　　　　　1.2 Provider的端口

　　　　　　1.3 Provider对外提供的接口、方法

　　　　　　1.4 dubbo版本号

　　　　　　1.5 访问Provider的协议

2.subscribe：订阅，当Consumer启动时，自动去Registry获取到所有已注册的服务信息

3.notify： 通知， 当Provider的信息发生变化时，自动由Registry向Consumer推送通知

4.invoke： 调用， Consumer调用Provider中的方法

　　　　　　4.1同步请求，消耗一定性能，但是必须是同步请求，因为需要接收调用方法后的结果

5.count： 次数， 每隔2分钟，Provider和Consumer自动向Monitor发送访问次数，Monitor进行统计

# dubbo的工作原理

**初始化过程细节**：第一步，就是将服务装载到容器中，然后准备注册服务。和Spring中启动过程类似，Spring启动时，将bean装载进容器中的时候，首先要解析bean。所以dubbo也是先读配置文件解析服务。

 

**解析服务**：1）、基于dubbo.jar内的META-INF/spring.handlers配置，spring在遇到dubbo名称空间时，会回调DubboNamespaceHandler类。

　　　　　2）、所有的dubbo标签都统一用DubboBeanDefinitionParser进行解析，基于一对一属性映射，将XML标签解析为Bean对象，生产者或者消费者初始化的时候，会将Bean对象转换为URL格式，将所有Bean属性转换成URL的参数。然后将URL传给Protocal扩展点，基于扩展点的Adaptive机制，根据URL的协议头，进行不同协议的服务暴露和引用。

 

**暴露服务**：1）、直接暴露服务端口：在没有注册中心的情况下，配置ServiceConfig解析出的URL，基于扩展点Adaptive机制，通过URL的协议头识别，直接调用DubboProtocol的export（）方法，打开服务端口

　　　　　2）、向注册中心暴露服务：将服务的IP和端口一同暴露给注册中心。ServiceConfig解析出的url格式，基于扩展点的Adaptive机制，通过URL的协议头识别，调用RegistryProtocol的export（）方法，将export参数中的提供者URL先注册到注册中心，再重新传给Protocol扩展点进行暴露。

 

**引用服务**：1）、直接引用服务：在没有注册中心的情况下，直连提供者，ReferenceConfig解析出URL格式，基于扩展点的Adaptive机制，通过URL协议头识别，直接调用DubboProtocol的refer方法，返回提供者引用。

　　　　　2）、从注册中心发现引用服务：ReferenceConfig解析出的URL的格式，基于扩展点的Adaptive机制，通过URL协议头识别，就会调用RegistryProtocol的refer方法，从refer参数总的条件，查询提供者URL，通过提供者URL协议头识别，就会调用DubboProtocol的refer（）方法，得到提供者引用。然后RegistryProtocol将多个提供者引用通过Cluster扩展点，伪装成单个提供者引用返回。

# 服务提供与消费详细过程

暴露服务的主过程：
首先ServiceConfig类拿到对外提供服务的实际类ref，然后将ProxyFactory类的getInvoker方法使用ref生成一个AbstractProxyInvoker实例，到这一步就完成具体服务到invoker的转化。接下来就是Invoker转换到Exporter的过程。 Dubbo处理服务暴露的关键就在Invoker转换到Exporter的过程，下面我们以Dubbo和rmi这两种典型协议的实现来进行说明： Dubbo的实现： Dubbo协议的Invoker转为Exporter发生在DubboProtocol类的export方法，它主要是打开socket侦听服务，并接收客户端发来的各种请求，通讯细节由dubbo自己实现。 Rmi的实现： RMI协议的Invoker转为Exporter发生在RmiProtocol类的export方法，他通过Spring或Dubbo或JDK来实现服务，通讯细节由JDK底层来实现。

服务消费的主过程：
首先ReferenceConfig类的init方法调用Protocol的refer方法生成Invoker实例。接下来把Invoker转为客户端需要的接口。