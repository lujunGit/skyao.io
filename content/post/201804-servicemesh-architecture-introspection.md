+++
title = "Service Mesh架构反思：proxy到底该做哪些事情？"

date = 2018-04-20
lastmod = 2018-04-20
draft = false

tags = ["ServiceMesh", "Istio", "Conduit"]
summary = "苦等一年，始终不见Istio的性能有实质性提升，失望之余，开始反思Istio而至Service Mesh的架构。焦点所在：proxy到底该做哪些事情？架构的优美和性能的实际表现该如何平衡？"
abstract = "苦等一年，始终不见Istio的性能有实质性提升，失望之余，开始反思Istio而至Service Mesh的架构。焦点所在：proxy到底该做哪些事情？架构的优美和性能的实际表现该如何平衡？"

[header]
image = "headers/post/201804-service-mesh-architecture-introspection.jpg"
caption = ""

+++

苦等一年，始终不见Istio的性能有实质性提升，失望之余，开始反思Istio而至Service Mesh的架构。焦点所在：proxy到底该做哪些事情？架构的优美和性能的实际表现该如何平衡？

## 问题所在

Istio的性能，从去年5月问世以来，就是一块心病，至少在我看来是如此。

前段时间Istio 0.7.1版本发布，不久我们得到了这个一个"喜讯"：Istio的性能在0.7.1版本中得到了大幅提升！

![](images/istio-performance-result.jpg)

最大QPS从0.5.1的700，到0.6.0的1000，再到0.7.1的1700，纵向比较Istio最近的性能提升的确不错，都2.5倍性能了。

但是，注意，QPS **700/1000/1700**。是的，没错，你没有看错，我也没有写错，我们并没有漏掉一个零或者两个零，就是这么高。

我稍微回顾一下近年中有实际感受的例子：

- 几个月之前写了一个基于netty4.1的HTTP转发的demo，没做任何优化和调教，轻松10万+的QPS
- 两年前写的dolphin框架，gRPC直连，15万QPS轻松做到
- 然后，白衣和隔壁老王两位同学轻描淡写的好像说过，OSP极致性能过了20万QPS了（背景补充：OSP是唯品会的服务化框架，sidecar模式，它的Proxy是被大家诟病以慢著称的Java）。

当然Istio测试场景稍微复杂一些，毕竟不是空请求，但是如论如何，从QPS十几二十万这个级别，直落到QPS一两千，中间有几乎100倍的差距。这个未免有些差距太大，从实用性上看，2000级别的QPS，实在低了点。

反思的第一个怀疑对象是proxy转发性能，但是随即否决：多了proxy肯定会有影响，但是远不至于此。而且，唯品会的OSP，还有华为的Service Mesher的性能表现也说明了问题不是出在proxy转发上。

> 注：关于proxy可能带来的性能开销，在之前的文章 [DreamMesh抛砖引玉(6)-性能开销](../201802-dreammesh-brainstorm-cost/) 一文中曾经有过详细的分析，可以参考。

## 背景回顾

Istio的性能问题其实由来已久，很早就暴露，官方也为此做了改进，我们现在看到的 700/1000/1700 已经是改进之后的结果。

今天来好好聊下这个问题，这涉及到Istio的架构设计，甚至涉及到Service Mesh的演进过程。

Service Mesh刚出现时，以Linkerd/Envoy为代表:

![](images/ppt-26.JPG)

这时的service mesh，主要是以proxy/sidecar的形式出现，功能基本都在sidecar中（linkerd有个named的模块分出了部分功能）。请求通过sidecar转发，期间发生的各种检查/判断/操作，都是在sidecar中进行。

![](images/ppt-8.JPG)

因此，从性能上说，与客户端/服务器端直接连接相比，做的事情基本一致，只是多了proxy转发的开销。而这个开销，与服务器端业务处理的开销相比，非常小，大多数情况下是可以忽略的。

然后Service Mesh演进到了第二代，Istio出现，和后面紧跟的Conduit。这个时候，在第一代的基础上，Service Mesh开始在对系统的掌控上发力，控制平面由此诞生：

![](images/ppt-29.JPG)

控制平面的加入，是一个非常令人兴奋的事情，因为带来了太多的实用型的功能，istio因此备受关注和推崇，随后紧跟Istio脚步的Conduit也沿用了这套架构体系：

![](images/ppt-31.JPG)

在这个架构当中，控制平面的三大模块，其中的Pilot和Auth都直接参与到traffic的处理流程，因此他们不会对运行时性能产生直接影响。

需要审视的是，需要参与到traffic流程的Mixer模块，请注意上图中，从envoy到mixer的两个箭头，这代表着两次调用，而且是两次远程调用。

下面给一个Mixer的单独架构图，可以更清楚的看到这一点：

![](images/mixer-traffic.svg)



