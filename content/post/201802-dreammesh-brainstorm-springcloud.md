+++
title = "DreamMesh抛砖引玉(8)-SpringCloud迁移"

date = 2018-02-24
lastmod = 2018-02-24
draft = true

tags = ["DreamMesh"]
summary = "Spring Cloud在未来很长一段时间之内都会是市场主流和很多公司的第一选择。如何在迁移到service mesh之前加强Spring Cloud，并为将来转入Service Mesh铺路，是一个艰难而极具价值的话题。"
abstract = "Spring Cloud在未来很长一段时间之内都会是市场主流和很多公司的第一选择。如何在迁移到service mesh之前加强Spring Cloud，并为将来转入Service Mesh铺路，是一个艰难而极具价值的话题。"

[header]
image = "headers/dreammesh-brainstorm-8.jpg"
caption = ""

+++

虽然Service Mesh号称下一代微服务，取代以Spring Cloud为典型代表的传统侵入式开发框架是Service Mesh与生俱来的天然目标之一。

但是以目前市场形式，Spring Cloud在未来很长一段时间之内都会是市场主流和很多公司的第一选择。如何在迁移到Service Mesh之前加强Spring Cloud，并为将来转入Service Mesh铺路，是一个艰难而极具价值的话题。

## 长期共存的大背景

首先我们将目标限定在准备进行互联网技术转型的传统企业用户，和部分技术能力还没有达到大型互联网企业水准的中小型互联网企业用户，这些企业有一个比较共性的地方：

1. 暂时还没有把微服务和容器等新兴技术玩转
2. 又有强烈的意愿往新技术方向做技术转型

在本系列前面的 [DreamMesh抛砖引玉(2)-CloudNative](201802-dreammesh-brainstorm-cloudnative/) 一文中我们对此有深入探讨。

其次，目前Service Mesh处于青黄不接的尴尬时期，暂时Istio和Conduit这两个寄予厚望的产品都还没有production ready，考虑到即使1.0发布，社区和用户也还有一个逐步接触，掌握的过程。而且很多公司出于谨慎也可能采取观望的姿态，Service Mesh的普及必然会需要时间。

可以参考微服务这几年的发展历程：

- 2014年出来微服务的概念
- 2015年国内开始讨论微服务，少数公司开始做尝试
- 2016/2017年很多公司陆陆续续在实践
- 2018年，还有很多公司没有上微服务，聚焦在传统企业，则可以说是大部分公司还出于初级阶段

类比之下，今天Service Mesh的在社区的知名度方面和2015年时微服务的状态类似。但是，有一个很大不同在于：微服务在2015年时在实践方面已经有很多公司已经实践并积累了足够的经验，包括类库，典型如Netflix和OSS套件，但是Service Mesh，尤其是以Istio和Conduit为代表的具备强大管理能力的Service Mesh，至今还没有落地实践可以参考。

因此，未来一段时间，必然会存在侵入式开发框架如Spring Cloud和Service Mesh并存的局面，也必然会有很多致力于微服务迁移的公司继续选择使用Spring Cloud。期间会有很长一段时间，Service Mesh和Spring Cloud并存。

### SpringCloud的不足之处

虽然Spring Cloud近两年风光无限，但是SpringCloud的不足还是非常明显，尤其服务治理功能太过薄弱。对比Istio的功能集合，Spring Cloud差距明显。

另外Spring Cloud的设计哲学是封装一套通用接口，然后理论上可以有多种实现，典型如服务发现的DiscoveryClient，可以有Eeurka/zookeeper/consul等诸多实现。但是Spring Cloud的封装过于简化，几乎是只支持最小功能集合，缺乏大量实用功能。

以DiscoveryClient接口为例，这是接口定义，方法屈指可数：

```java
public interface DiscoveryClient {

    /**
     * 实现可读的描述，用于HealthIndicator
     */
    String description();

    /**
     * 获取和指定serviceId关联的所有服务实例
     */
    List<ServiceInstance> getInstances(String serviceId);

    /**
     * 返回所有已知的服务id。（注意只是id）
     */
    List<String> getServices();
}
```

最关键的`getInstances(String serviceId);`，入口参数只有一个serviceId。再看
ServiceInstance的定义：

| 参数 | 类型 | 说明 |
|--------|--------|--------|
|    serviceId    |    String    |   service id，由DiscoveryClient设置     |
|    host    |    String    |    主机名    |
|    port    |    int    |    端口    |
|    secure    |    boolean    |    是否加密，简单说就是是否https    |
|    metadata    |    `Map<String, String>`    |    元数据    |

和一个成熟的服务注册机制（如原生的Euraka/Consul）相比，缺少诸多功能：

- 没有version参数，所以无法实现语义化版本。
- 没有group或者namespace参数，所以无法为服务分组，当服务数量多时管理困难。
- 没有zone/datacenter参数，所以无法实现多机房部署
- 服务没有status参数，无法支持特殊状态例如consul的"maintainance"。注：ServiceRegistry接口上又有setStatus()。
- 接口上也没有定义监听方法，只能交给具体实现。

> 备注：好在还有一个metadata参数，总算留了一个口子，可以自行扩展以实现部分功能。但是需要自己实现。

除了服务注册和服务发现外，Spring Cloud目前严重依赖Ribbon来实现客户端负载均衡，而Ribbon是不支持权重的，因此类似"切1%的流量去某个实例"这样的典型灰度需求在Spring Cloud下是一筹莫展。

当然，我们今天不是来声讨Spring Cloud的。

## 后记

有兴趣的朋友，请联系我的微信，加入DreamMesh内部讨论群。
