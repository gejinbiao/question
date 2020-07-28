# dubbo详解
## 基础概念

Dubbo就是SOA服务治理方案的核心框架。用于分布式调用，其重点在于分布式的治理。

Dubbo是Alibaba开源的分布式服务框架，它最大的特点是按照分层的方式来架构，使用这种方式可以使各个层之间解耦合（或者最大限度地松耦合），比如表现层和业务层就需要解耦合。

从面向服务的角度来看，Dubbo采用的是一种非常简单的模型，要么是提供方提供服务，要么是消费方消费服务，所以基于这一点可以抽象出服务提供方（Provider）和服务消费方（Consumer）两个角色。

除了以上两个角色，它还有注册中心和监控中心。它可以通过注册中心对服务进行注册和订阅；可以通过监控中心对服务进行监控，这样的话，就可以知道哪些服务使用率高、哪些服务使用率低。对使用率高的服务增加机器，对使用率低的服务减少机器，达到合理分配资源的目的。


dubbo是一个高性能轻量级开源的RPC框架。主要功能分为三点：

1. 提供面向接口的远程方法调用
1. 负载均衡和智能容错
1. 以及服务的自动注册和发现

# Dubbo核心功能
1. Remoting：远程通讯，提供对多种NIO框架抽象封装，包括“同步转异步”和“请求-响应”模式的信息交换方式。
2. Cluster：服务框架，提供基于接口方法的透明远程过程调用，包括多协议支持，以及软负载均衡，失败容错，地址路由，动态配置等集群支持。
3. Registry：服务注册，基于注册中心目录服务，使服务消费方能动态的查找服务提供方，使地址透明，使服务提供方可以平滑增加或减少机器。

# 什么是RPC及其原理
- RPC:部署在不同的机器的服务需要进行方法调用，通过RPC具体的实现则可以完成远程过程的调用，而不需要具体了解协议
- 原理：在客户端的底层和服务端的底层分别进行封装，客户端调用客户端的封装模块，找到服务端的地址，并进行协议的封装，调用服务端封装模块，服务端封装模块调用本地服务，并进行返回，客户端封装模块进行封装返回给客户端

# dubbo相对于http请求的好处
1. 负载均衡，可以将统一服务部署不同机器
1. 服务注册和发现，能够高效地找到需要的服务地址
1. 服务降级的功能，能够对服务进行容错
1. 可以监控服务的调用状态，根据统计情况，对服务进行治理

# 分布式
- 分布式：将应用根据不同功能拆分成不同服务
- 分布式好处：1.团队开发效率更高，每个小团队负责一个服务；2.服务切分后，更好扩展和维护及复用；3.分布式部署后能够更好地治理和分配资源；4.降低组件的复杂度，提高运行效率，增加可靠性和容错性；
- 分布式缺点：1.学习成本增加；2.服务间调用的网络传输损耗；3.数据传输的安全性问题；4.故障排除定位困难；

# Dubbo组件角色
- Container:运行服务的容器；
1. 负责启动加载提供者
- Registry:注册中心；
1. 接受注册和订阅；
2. 当服务有变更时，基于长连接将变更数据推送给消费者；
- Provider:服务提供者；
1. 向服务注册中心注册服务,暴露自己的服务;
2. 内存中累计调用次数和调用时间，定时每分钟将统计数据发送给监控中心；
- Consumer:服务消费者；
1. 向服务注册中心订阅服务，并将服务的地址注册表缓存本地；
2. 服务调用时，根据本地缓存注册表进行服务调用；
3. 服务调用时，会根据负载均衡策略选择服务调用；
4. 服务调用时，有容错机制的保障;
5. 内存中累计调用次数和调用时间，定时每分钟将统计数据发送给监控中心；
- Monitor:
1. 统计服务调用的调用次数和调用时长；

![NNobVg.png](https://s1.ax1x.com/2020/06/23/NNobVg.png)
调用关系说明：

1. 服务容器负责启动，加载，运行服务提供者。
1. 服务提供者在启动时，向注册中心注册自己提供的服务。
1. 服务消费者在启动时，向注册中心订阅自己所需的服务。
1. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
1. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
1. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心Monitor。

# dubbo发布和消费的过程

- dubbo发布：
1. 根据发布的接口（服务配置）通过ProxyFactory类的getInvoker生成一个AbstractProxyInvoker实例
2. 通过Invoker转换到Exporter，将服务根据特定的协议对外开放.

![NNHJDf.png](https://s1.ax1x.com/2020/06/23/NNHJDf.png)

- dubbo订阅：
1. 根据订阅的接口（引用配置），根据特定的协议生成相应的Invoker
2. 通过ProxyFactory将Invoker生成代理的引用
3. 根据代理对象进行方法的调用


![NNH6bT.png](https://s1.ax1x.com/2020/06/23/NNH6bT.png)

# dubbo容错

- 原理：在调用过程中根据配置选择对应的容错对象，进行调用
- 容错模式：
	- 重试(Failover Cluster)：根据配置次数进行重试，默认2次；用于读操作或者幂等写操作；
	- 快速失败(Failfast Cluster)：失败立即报错；用于非幂等写操作；
	- 失败安全(Failsafe Cluster)：异常直接忽略；如写审计日志；
	- 失败自动恢复(Failback Cluster)：后台记录失败请求，后期进行重试；用于通知操作；
	- 并行(Forking Cluster):并发调用多个服务，一个成功及返回；用于具有对实时性读有要求的场景，但浪费资源；
	- 广播(Broadcast Cluster)：逐个调用所有服务，一台报错则全部报错；用于通知所有提供者更新本地缓存或日志等信息；
-实现：
	-提供方：
	<dubbo:service cluster="failsafe" />

- 消费方：

	<dubbo:reference cluster="failsafe" />

# dubbo负载均衡策略

1. 随机(Random LoadBalance)：基于权重随机选择调用
2. 权重轮询(RoundRobin LoadBalance)：基于权重轮流调用
3. 最少活跃数(LeastActive LoadBalance)：活跃数指服务提供者调用前后的调用时长，也就是根据服务提供者的性能进行调用
4. 一致性hash(ConsistentHash LoadBalance)：对请求参数求hash值，根据hash值调用到相应的服务提供者，好处是同类请求能够在一个服务节点响应
# dubbo常用协议

- Dubbo：Dubbo 缺省协议采用单一长连接和 NIO 异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。
- RMI：RMI 协议采用 JDK 标准的 java.rmi.* 实现，采用阻塞式短连接和 JDK 标准序列化方式。
- Hessian：Hessian 协议用于集成 Hessian 的服务，Hessian 底层采用 HTTP 通讯，采用 Servlet 暴露服务，Dubbo 缺省内嵌 Jetty 作为服务器实现。
- HTTP：采用 Spring 的 Http Invoker 实现。
- Webservice：基于 CXF 的 frontend-simple 和 transports-http 实现。

# Dubbo总体架构
上面介绍给出的都是抽象层面的组件关系，可以说是纵向的以服务模型的组件分析，其实Dubbo最大的特点是按照分层的方式来架构，使用这种方式可以使各个层之间解耦合（或者最大限度地松耦合）。所以，我们横向以分层的方式来看下Dubbo的架构，如图所示：

![NNb0oD.png](https://s1.ax1x.com/2020/06/23/NNb0oD.png)

Dubbo框架设计一共划分了10个层，而最上面的Service层是留给实际想要使用Dubbo开发分布式服务的开发者实现业务逻辑的接口层。图中左边淡蓝背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口， 位于中轴线上的为双方都用到的接口。

下面，结合Dubbo官方文档，我们分别理解一下框架分层架构中，各个层次的设计要点：

1. 服务接口层（Service）：与实际业务逻辑相关的，根据服务提供方和服务消费方的 业务设计对应的接口和实现。
2. 配置层（Config）：对外配置接口，以ServiceConfig和ReferenceConfig为中心，可以直接new配置类，也可以通过Spring解析配置生成配置类。
3. 服务代理层（Proxy）：服务接口透明代理，生成服务的客户端Stub和服务器端Skeleton，以ServiceProxy为中心，扩展接口为ProxyFactory。
4. 服务注册层（Registry）：封装服务地址的注册与发现，以服务URL为中心，扩展接口为RegistryFactory、Registry和RegistryService。可能没有服务注册中心，此时服务提供方直接暴露服务。
5. 集群层（Cluster）：封装多个提供者的路由及负载均衡，并桥接注册中心，以Invoker为中心，扩展接口为Cluster、Directory、Router和LoadBalance。将多个服务提供方组合为一个服务提供方，实现对服务消费方来透明，只需要与一个服务提供方进行交互。
6. 监控层（Monitor）：RPC调用次数和调用时间监控，以Statistics为中心，扩展接口为MonitorFactory、Monitor和MonitorService。
7. 远程调用层（Protocol）：封将RPC调用，以Invocation和Result为中心，扩展接口为Protocol、Invoker和Exporter。Protocol是服务域，它是Invoker暴露和引用的主功能入口，它负责Invoker的生命周期管理。Invoker是实体域，它是Dubbo的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起invoke调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。
8. 信息交换层（Exchange）：封装请求响应模式，同步转异步，以Request和Response为中心，扩展接口为Exchanger、ExchangeChannel、ExchangeClient和ExchangeServer。
9. 网络传输层（Transport）：抽象mina和netty为统一接口，以Message为中心，扩展接口为Channel、Transporter、Client、Server和Codec。
10. 数据序列化层（Serialize）：可复用的一些工具，扩展接口为Serialization、 ObjectInput、ObjectOutput和ThreadPool。
从上图可以看出，Dubbo对于服务提供方和服务消费方，从框架的10层中分别提供了各自需要关心和扩展的接口，构建整个服务生态系统（服务提供方和服务消费方本身就是一个以服务为中心的）。

根据官方提供的，对于上述各层之间关系的描述，如下所示：
1. 在RPC中，Protocol是核心层，也就是只要有Protocol + Invoker + Exporter就可以完成非透明的RPC调用，然后在Invoker的主过程上Filter拦截点。

2. 图中的Consumer和Provider是抽象概念，只是想让看图者更直观的了解哪些分类属于客户端与服务器端，不用Client和Server的原因是Dubbo在很多场景下都使用Provider、Consumer、Registry、Monitor划分逻辑拓普节点，保持概念统一。

3. 而Cluster是外围概念，所以Cluster的目的是将多个Invoker伪装成一个Invoker，这样其它人只要关注Protocol层Invoker即可，加上Cluster或者去掉Cluster对其它层都不会造成影响，因为只有一个提供者时，是不需要Cluster的。

4. Proxy层封装了所有接口的透明化代理，而在其它层都以Invoker为中心，只有到了暴露给用户使用时，才用Proxy将Invoker转成接口，或将接口实现转成Invoker，也就是去掉Proxy层RPC是可以Run的，只是不那么透明，不那么看起来像调本地服务一样调远程服务。
5. 而Remoting实现是Dubbo协议的实现，如果你选择RMI协议，整个Remoting都不会用上，Remoting内部再划为Transport传输层和Exchange信息交换层，Transport层只负责单向消息传输，是对Mina、Netty、Grizzly的抽象，它也可以扩展UDP传输，而Exchange层是在传输层之上封装了Request-Response语义。

6. Registry和Monitor实际上不算一层，而是一个独立的节点，只是为了全局概览，用层的方式画在一起。

# 服务调用流程

![NNqP61.png](https://s1.ax1x.com/2020/06/23/NNqP61.png)

# 连接方式

1、直连：

不通过注册中心，直接由消费者访问提供者。
2、集群：

提供者把服务注册到注册中心，然后消费者询问注册中心，请求对应的服务需要请求哪个提供者，注册中心返回结果，消费者根据结果向提供者请求服务。
3、超时和重连机制：

（1）dubbo超时，不会返回结果，直接报异常
（2）配置超时重试次数、超时时间：
    ```
        <!-- 服务调用超时设置为5秒,超时不重试-->
        <dubbo:service interface="com.provider.service.DemoService" ref="demoService"  retries="0" timeout="5000"/>
    ```
（3）dubbo在调用服务不成功时，默认会重试2次。
    Dubbo的路由机制，会把超时的请求路由到其他机器上，而不是本机尝试，所以 dubbo的重试机器也能一定程度的保证服务的质量。但是如果不合理的配置重试次数，当失败时会进行重试多次，这样在某个时间点出现性能问题，调用方再连续重复调用，系统请求变为正常值的retries倍，系统压力会大增，容易引起服务雪崩，需要根据业务情况规划好如何进行异常处理，何时进行重试。

# group及version

1. group：用于对服务进行隔离，这里可以实现灰度功能的作用。
2. version：当一个接口的实现，出现不兼容升级时，可以用版本号过渡，版本号不同的服务相互间不引用

# 问题（走过的坑）
1、Dubbo调用接口，该接口抛出的运行时异常，在调用函数里面无法捕获的
2、一个经典问题： Forbid consumer 127.0.0.1 access service com.mall.api.service.MallService from registry localhost:2181 use dubbo version 2.9.2-SNAPSHOT, Please check registry access list (whitelist/blacklist).

这个错误一般表面上说禁止消费服务，实际上开发人员并没有做任何限制。其实，这个问题一般由两种情况引起的
（1）第一种情况：没有provider提供对应的服务
（2）第二种情况：消费者消费的服务的“版本号 version”和生产者提供的服务的“版本号 version”不一致，或者没有显式说明版本号。

3、当dubbo超时，返回的结果会为null
4、dubbo如果报异常了，dubbo不会返回错误码，因此需要在对外提供的接口处利用参数传递错误码、错误信息
5、在dubbo的provider和consumer的配置文件中，如果都配置了timeout的超时时间，dubbo默认以consumer中配置的时间为准

#面试题相关
## 为什么要用Dubbo？
使用 Dubbo 可以将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，可用于提高业务复用灵活扩展，使前端应用能更快速的响应多变的市场需求。

下面这张图可以很清楚的诠释，最重要的一点是，分布式架构可以承受更大规模的并发流量。
![NNOSMR.png](https://s1.ax1x.com/2020/06/23/NNOSMR.png)
下面是 Dubbo 的服务治理图。
![NNOVRH.png](https://s1.ax1x.com/2020/06/23/NNOVRH.png)

## Dubbo 和 Spring Cloud 有什么区别？
两个没关联，如果硬要说区别，有以下几点。

1）通信方式不同

Dubbo 使用的是 RPC 通信，而 Spring Cloud 使用的是 HTTP RESTFul 方式。

2）组成部分不同

![NNOnsI.png](https://s1.ax1x.com/2020/06/23/NNOnsI.png)

## dubbo都支持什么协议，推荐用哪种？

- dubbo://（推荐）

- rmi://

- hessian://

- http://

- webservice://

- thrift://

- memcached://

- redis://

- rest://

## Dubbo需要 Web 容器吗？
不需要，如果硬要用 Web 容器，只会增加复杂性，也浪费资源。
## Dubbo内置了哪几种服务容器？
- Spring Container

- Jetty Container

- Log4j Container
Dubbo 的服务容器只是一个简单的 Main 方法，并加载一个简单的 Spring 容器，用于暴露服务。
## 画一画服务注册与发现的流程图

![NNOWex.png](https://s1.ax1x.com/2020/06/23/NNOWex.png)
## Dubbo默认使用什么注册中心，还有别的选择吗？
推荐使用 Zookeeper 作为注册中心，还有 Redis、Multicast、Simple 注册中心，但不推荐

##Dubbo有哪几种配置方式？
1）Spring 配置方式
2）Java API 配置方式

## Dubbo 核心的配置有哪些？

![NNXMc9.png](https://s1.ax1x.com/2020/06/23/NNXMc9.png)

配置之间的关系见下图。

![NNX1n1.png](https://s1.ax1x.com/2020/06/23/NNX1n1.png)

## 在 Provider 上可以配置的 Consumer 端的属性有哪些？
1）timeout：方法调用超时
2）retries：失败重试次数，默认重试 2 次
3）loadbalance：负载均衡算法，默认随机
4）actives 消费者端，最大并发调用限制

## Dubbo启动时如果依赖的服务不可用会怎样？
Dubbo 缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止 Spring 初始化完成，默认 check="true"，可以通过 check="false" 关闭检查。
## Dubbo推荐使用什么序列化框架，你知道的还有哪些？
推荐使用Hessian序列化，还有Duddo、FastJson、Java自带序列化。
##Dubbo默认使用的是什么通信框架，还有别的选择吗？
Dubbo 默认使用 Netty 框架，也是推荐的选择，另外内容还集成有Mina、Grizzly。

## Dubbo有哪几种集群容错方案，默认是哪种？
![NNjpE6.png](https://s1.ax1x.com/2020/06/23/NNjpE6.png)

## Dubbo有哪几种负载均衡策略，默认是哪种？
![NNjAvd.png](https://s1.ax1x.com/2020/06/23/NNjAvd.png)

## 注册了多个同一样的服务，如果测试指定的某一个服务呢？
可以配置环境点对点直连，绕过注册中心，将以服务接口为单位，忽略注册中心的提供者列表。

## Dubbo支持服务多协议吗？
Dubbo 允许配置多协议，在不同服务上支持不同协议或者同一服务上同时支持多种协议。

## 当一个服务接口有多种实现时怎么做？

当一个接口有多种实现时，可以用 group 属性来分组，服务提供方和消费方都指定同一个 group 即可。

## 服务上线怎么兼容旧版本？
可以用版本号（version）过渡，多个不同版本的服务注册到注册中心，版本号不同的服务相互间不引用。这个和服务分组的概念有一点类似。

## Dubbo可以对结果进行缓存吗？
可以，Dubbo 提供了声明式缓存，用于加速热门数据的访问速度，以减少用户加缓存的工作量。

## Dubbo服务之间的调用是阻塞的吗？
默认是同步等待结果阻塞的，支持异步调用。

Dubbo 是基于 NIO 的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销较小，异步调用会返回一个 Future 对象。

异步调用流程图如下。
![NNvEJU.png](https://s1.ax1x.com/2020/06/23/NNvEJU.png)

## Dubbo支持分布式事务吗？
目前暂时不支持，后续可能采用基于 JTA/XA 规范实现，如以图所示。
![NNvDFf.png](https://s1.ax1x.com/2020/06/23/NNvDFf.png)

## Dubbo支持服务降级吗？
Dubbo 2.2.0 以上版本支持。

## 服务提供者能实现失效踢出是什么原理？
服务失效踢出基于 Zookeeper 的临时节点原理。

## 如何解决服务调用链过长的问题？
Dubbo 可以使用 Pinpoint 和 Apache Skywalking(Incubator) 实现分布式服务追踪，当然还有其他很多方案。

## 服务读写推荐的容错策略是怎样的？
读操作建议使用 Failover 失败自动切换，默认重试两次其他服务器。

写操作建议使用 Failfast 快速失败，发一次调用失败就立即报错。

## Dubbo的管理控制台能做什么？
管理控制台主要包含：路由规则，动态配置，服务降级，访问控制，权重调整，负载均衡，等管理功能。

## 说说 Dubbo 服务暴露的过程。
Dubbo 会在 Spring 实例化完 bean 之后，在刷新容器最后一步发布 ContextRefreshEvent 事件的时候，通知实现了 ApplicationListener 的 ServiceBean 类进行回调 onApplicationEvent 事件方法，Dubbo 会在这个方法中调用 ServiceBean 父类 ServiceConfig 的 export 方法，而该方法真正实现了服务的（异步或者非异步）发布。

## 你还了解别的分布式框架吗？
别的还有 Spring cloud、Facebook 的 Thrift、Twitter 的 Finagle 等。

## Dubbo 能集成 Spring Boot 吗？
可以的，项目地址如下。

https://github.com/apache/incubator-dubbo-spring-boot-project

## 在使用过程中都遇到了些什么问题？
Dubbo 的设计目的是为了满足高并发小数据量的 rpc 调用，在大数据量下的性能表现并不好，建议使用 rmi 或 http 协议。

## 你觉得用 Dubbo 好还是 Spring Cloud 好？

扩展性的问题，没有好坏，只有适合不适合，不过我好像更倾向于使用 Dubbo, Spring Cloud 版本升级太快，组件更新替换太频繁，配置太繁琐，还有很多我觉得是没有 Dubbo 顺手的地方……
