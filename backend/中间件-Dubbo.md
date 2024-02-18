##### RPC是什么东西？

RPC是 remote procedure call 远程过程调用，RPC是一种协议。解决的问题主要是分布式环境下服务调用。单机时代，A function调用B function只需要有方法指针即可在内存中直接调用，而RPC解决的是 两台服务上 A fun调用B fun的解决办法。所以这里基本就涉及到几点。

- 网络传输协议：tcp、http
- 函数映射：比如A想调用B方法，要传入什么才能表示调用B方法，比如gRpc里就直接使用字符串，dubbo里通过zk，consumer，provioder、zk也是实现了这套语义
- 序列化协议：参数、结果都是需要一个统一的格式化方式以后，再传递给网络协议。

以上三个就是我自己理解的RPC最最基本的 三个要素。拿dubbo来举例来说， 网络传输层使用 netty（底层tcp）。函数映射，dubbo 通过zk，consumer  provioder 逻辑概念，实现了 A方法对B方法的引用，序列化协议，dubbo默认使用 hessian 序列化。

##### Dubbo 原理架构 是什么 ？

dubbo是一个高性能的RPC框架。主要是随着服务化的进一步发展，服务越来越多，之间的调用关系也是越来越复杂，所以越来越需要规范服务之间的调用体系，以及针对服务调用进行治理。

Dubbo对于RPC概念抽象出了  服务提供方 和 服务消费方，注册中心，监控中心等概念。简单来说就是 在容器启动的时候，dubbo通过配置文件，在配置管理中心上 注册了 服务提供方以及服务消费方，消费方通过监控 提供方的地址，使用负载均衡算法，底层使用dubbo封装的网络层通信调用，实现了高性能的RPC服务。

![CleanShot 2023-03-21 at 03.23.23@2x](/Users/jazz/Library/Application Support/CleanShot/media/media_fBE4bjxs2J/CleanShot 2023-03-21 at 03.23.23@2x.png)



##### Dubbo分为哪几个模块？

- **dubbo-common:公共逻辑模块**: 包括Util类和通用模型
- **dubbo-remoting 远程通信模块**: 相当于dubbo协议的实现，如果RPC使用RMI协议则不需要使用此包
- **dubbo-rpc 远程调用模块**: 抽象各种协议，以及动态代理，包含一对一的调用，不关心集群的原理。
- **dubbo-cluster 集群模块**: 将多个服务提供方伪装成一个提供方,包括负载均衡,容错,路由等,集群的地址列表可以是静态配置的,也可以是注册中心下发的.
- **dubbo-registry 注册中心模块**: 基于注册中心下发的集群方式,以及对各种注册中心的抽象
- **dubbo-monitor 监控模块**: 统计服务调用次数,调用时间,调用链跟踪的服务.
- **dubbo-config 配置模块**: 是dubbo对外的api,用户通过config使用dubbo,隐藏dubbo所有细节
- **dubbo-container 容器模块**: 是一个standlone的容器,以简单的main加载spring启动,因为服务通常不需要Tomcat/Jboss等web容器的特性,没必要用web容器去加载服务.

##### Dubbo启动/调用过程是什么？

1. 服务提供者启动，开启Netty服务，创建Zookeeper客户端，向注册中心注册服务。

2. 服务消费者启动，通过Zookeeper向注册中心获取服务提供者列表，与服务提供者通过Netty建立长连接。

3. 服务消费者通过接口开始远程调用服务，ProxyFactory通过初始化Proxy对象，Proxy通过创建动态代理对象。

4. 动态代理对象通过invoke方法，层层包装生成一个Invoker对象，该对象包含了代理对象。

5. Invoker通过路由，负载均衡选择了一个最合适的服务提供者，在通过加入各种过滤器，协议层包装生成一个新的DubboInvoker对象。

6. 再通过交换成将DubboInvoker对象包装成一个Reuqest对象，该对象通过序列化通过NettyClient传输到服务提供者的NettyServer端。

7. 到了服务提供者这边，再通过反序列化、协议解密等操作生成一个DubboExporter对象,再层层传递处理,会生成一个服务提供端的Invoker对象.

8. 这个Invoker对象会调用本地服务，获得结果再通过层层回调返回到服务消费者，服务消费者拿到结果后，再解析获得最终结果。

##### Dubbo序列化框架是?

hessian，java二进制，json等  hessian是默认协议。

##### Dubbo使用的通信框架？

默认netty，也可以使用mina

##### Dubbo集群容错方案

在Dubbo设计中，通过Cluster这个接口的抽象，把一组可供调用的Provider信息组合成为一个统一的`Invoker`供调用方进行调用。经过路由规则过滤，负载均衡选址后，选中一个具体地址进行调用，如果调用失败，则会按照集群配置的容错策略进行容错处理。

Dubbo默认内置了若干容错策略，如果不能满足用户需求，则可以通过自定义容错策略进行配置。

内置容错策略

Dubbo主要内置了如下几种策略：

- Failover(失败自动切换)：随机选一个其他可用地址，重试，默认重试2次。
- Failsafe(失败安全)：返回空，记录日志
- Failfast(快速失败)：报错，调用失败
- Failback(失败自动恢复)：加入异步列表，额外线程重试，原来的调用方也无感知的，适用于 触发类的方法
- Forking(并行调用)：一开始就并发调用
- Broadcast(广播调用)：所有提供者 轮询调用，有一个失败就算失败

##### Dubbo服务降级，失败重试怎么做？

可以通过 dubbo:reference 中设置 mock="return null"。mock 的值也可以修改为 true，然后再跟接口同一个路径下实现一个 Mock 类，
命名规则是 “接口名称+Mock” 后缀。然后在 Mock 类里实现自己的降级逻辑

##### Dubbo配置文件怎么加载到spring里的

Spring 容器在启动的时候，会读取到 Spring 默认的一些 schema 以及 Dubbo 自定义的 schema，每个 schema 都会对应一个自己的
NamespaceHandler，NamespaceHandler 里面通过 BeanDefinitionParser 来解析配置信息并转化为需要加载的 bean 对象!

##### Dubbo spi 和 JDK spi区别

Java SPI：Service Provider Interface 一个**接口**多种实现，通过配置确定使用哪个实现。

Dubbo SPI：Java SPI 增强版，但并不是基于 Java SPI 去实现的，而是 Dubbo 自己按照 Java SPI 的功能，重写了一次，并增加了扩展点 IOC 和 AOP 的支持。

##### Dubbo 怎么实现结果缓存？

```xml
<dubbo:reference interface="com.foo.DemoService" cache="lru" />
```

目前Dubbo3.0版本及高于其的版本都支持以下几种内置的缓存策略：

- `lru` 基于最近最少使用原则删除多余缓存，保持最热的数据被缓存。
- `lfu`基于淘汰使用频次最低的原则来实现缓存策略。
- `expiring`基于过期时间原则来实现缓存策略。
- `threadlocal` 当前线程缓存，比如一个页面渲染，用到很多 portal，每个 portal 都要去查用户信息，通过线程缓存，可以减少这种多余访问。
- `jcache` 与 [JSR107](http://jcp.org/en/jsr/detail?id=107') 集成，可以桥接各种缓存实现。
