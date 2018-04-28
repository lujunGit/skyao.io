+++
title = "Mixer Cache: Istio的阿克琉斯之踵"

date = 2018-04-27
lastmod = 2018-04-27
draft = false

tags = ["Istio"]
summary = "为了架构的优雅，Istio设计了Mixer，将大量的功能从Sidecar中搬了出来。为了减少Mixer远程调用带来的性能，又精心设计了一套异常复杂的缓存。然而，这个Mixer Cache，却有一个致命之处......"

abstract = "为了架构的优雅，Istio设计了Mixer，将大量的功能从Sidecar中搬了出来。为了减少Mixer远程调用带来的性能，又精心设计了一套异常复杂的缓存。然而，这个Mixer Cache，却有一个致命之处......"

[header]
image = "headers/post/201804-istio-achilles-heel.jpg"
caption = ""

+++

## 前情回顾

在我的上一个博客文章中，出于对性能的担心，我和大家探讨并反思了Service Mesh的架构。关注的焦点在于mixer的职责和设计的初衷，以及由此带来的问题：

[Service Mesh架构反思：数据平面和控制平面的界线该如何划定？](../201804-servicemesh-architecture-introspection/)

> 注：推荐在阅读本文之前先阅读这个文章，以便对mixer有基本的了解。

期间，对于Mixer Cache我存在几个质疑，由于官方文档和网上资料都几乎找不到任何相关的详细信息。因此，我不得不通过阅读源代码的方式来深入了解细节。

而从现在分析的情况看，Istio的这个mixer cache实现有些复杂，设计精巧或者说不太容易理解。令人担忧的是，存在隐患导致在特定场景会完全失效。

这篇文档，将从mixer的工作原理，缓存的设计开始。

## Mixer缓存工作原理

我们先看看mixer是如何工作的， 简单的说，是envoy从每次请求中获取信息，然后发起两次对mixer的请求：

1. 在转发请求之前：这时需要做前提条件检查和配额管理，只有满足条件的请求才会做转发
2. 在转发请求之后：这时要上报日志等，术语上称为遥感信息，**Telemetry**，或者**Reporting**。

### Check方法介绍

我们的焦点落在转发之前的这次请求，称为Check方法，方法如下：

```properties
rpc Check(CheckRequest) returns (CheckResponse)
```

请求 `CheckRequest` 的内容如下（其他字段暂时忽略）：

| 字段            | 类型                                     | 描述                                                         |
| --------------- | ---------------------------------------- | ------------------------------------------------------------ |
| attributes      | CompressedAttribute                      | 用于此次请求的属性。<br> mixer的配置决定这些属性将被如何使用以创建在应答中返回的结果。 |

attributes 属性是envoy从请求中提取出来的，其内容类似如下：

```properties
request.path: xyz/abc
request.size: 234
request.time: 12:34:56.789 04/17/2017
source.ip: 192.168.0.1
target.service: example
```

Mixer中的Adapter将根据这些 attributes 属性来进行判断和处理，比如进行前置条件检查。然后将结果发送回envoy。简单起见，我们只看前置条件检查相关的内容。应答返回名为precondition的字段，表示前置条件检查的结果，具体如下：

| 字段                 | 类型                                                         | 描述                                                         |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| status               | [google.rpc.Status](#google.rpc.Status)                      | 状态码OK表示所有前置条件均满足。任何其它状态码表示不是所有的前置条件都满足，并且在detail中描述为什么。 |
| validDuration        | [google.protobuf.Duration](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#duration) | 时间量，在此期间这个结果可以认为是有效的                     |
| validUseCount        | int32                                                        | 可使用的次数，在此期间这个结果可以认为是有效的               |
| attributes           | CompressedAttributes                                         | mixer返回的属性。<br>返回的切确属性集合由mixer配置的adapter决定。这些属性用于传送新属性，这些新属性是Mixer根据输入的属性集合和它的配置派生的。 |
| referencedAttributes | ReferencedAttributes                                         | 在匹配条件并生成结果的过程中使用到的全部属性集合。           |

从Check方法的输入输出，我们可以看到：

1. 前置条件检查输入主要是attributes字段，这是envoy提取的属性列表，注意此时envoy是没有办法得知mixer中的adapter到底会关心哪些属性，因此envoy只能选择将所有属性都发送给mixer
2. 前置条件检查的输出中，status代表检查结果，理所当然的应该出现。validDuration和validUseCount是对缓存的限制，attributes传递新属性。唯独，referencedAttributes的出现有些奇怪，而这个referencedAttributes的设计，则是本次讨论的焦点。

### referencedAttributes的特殊设计

referencedAttributes是mixer中的adapter在做条件匹配并生成结果的过程中使用到的全部属性集合。

为什么要有这么一个设计呢？我们来看mixer请求过程中输入情况：我们假定当前mixer中有三个生效的adapter，其中每个adapter的逻辑都和特定的属性相关。

![](images/referencedAttributes.jpg)

再假设envoy在处理traffic请求时，从请求中提取出5个属性，这五个属性中a/b/c是三个adapter分别使用到的属性，然后e/f两个属性当前没有adapter使用（实际情况属性会远比这个复杂，提取的属性有十几二十，其中有更多的属性不被adapter使用）。

在这个请求应答模型中，注意envoy是不能提前知道adaper可能要哪些属性的，因此只能选择全部属性提交。按照通常的缓存设计思路，我们应该将输入作为key，输出作为value。但是，如果我们将“a=1,b=2,c=3,e=0,f=0”作为key时，我们会发现，当e/f两个属性发生变化时，产生的新key “a=1,b=2,c=3,e=1,f=1”/“a=1,b=2,c=3,e=3,f=4”/“a=1,b=2,c=3,e=10,f=30” 对应的value会大量的重复，而这些属性值的变化对于adapter来说完全没有意义：e/f两个属性根本不被adapter使用。

因此，Istio在设计mixer cache时，选择了一个特殊方式：

1. 在mixer的response中，返回adapter使用到的属性名，在这个例子中就是"a/b/c"。
2. envoy在收到response之后，检查这个使用的属性列表，发现只有"a/b/c"三个属性被adapter使用
3. envoy在缓存这个应答结果时，会选择将属性列表简化为只有adapter使用的属性，如“a=1,b=2,c=3”
4. 因此mixer在构建缓存项时，可以就

这些被adapter使用的属性在Check 方法的response中以 referencedAttributes 字段表示。

这样缓存的key的数量就大为减少，否者每次提交的key有十几二十个，有些属性的值还每次都变化如request.id。如果不进行这样的优化，则缓存根本无从谈起。

### 缓存的保存和查找方式

经过 referencedAttributes 字段的优化之后，输入的属性就被简化为“a=1,b=2,c=3”，然后envoy将保存这个输入到缓存中：

1. envoy将记录mixer关注"a/b/c"这样一组属性组合（注意会不止一组）
2. envoy将“a=1,b=2,c=3”这个实际被使用的属性值进行签名（理解为某种hash算法）得到缓存的key

缓存保存之后，envoy在处理后面的请求，如“a=1,b=2,c=3,e=1,f=1”/“a=1,b=2,c=3,e=3,f=4”/“a=1,b=2,c=3,e=10,f=30” 这三个请求时，就会尝试从缓存中查找：

1. envoy会先根据保存的被关注属性组合，看请求是否命中，比如这里的"a/b/c"属性组合就可以匹配这三个请求。
2. 然后根据"a/b/c"组合简化请求的属性为“a=1,b=2,c=3”，再进行签名计算
3. 然后再以计算的来的签名为key在缓存中查找。
4. 如果找到，返回缓存结果。如果没有找到，继续发送请求到mixer，然后保存得到的response到缓存中

这就是mixer cache的工作原理，而实际上，mixer cache的实现细节远比这里描述的复杂，有很多细节如absence key，有效时间，有效使用次数，匹配方式的优化。理论上说，有了这么一个明显是精心设计的mixer cache的加持，Istio中mixer和sidecar分离造成的性能问题得以解决，而mixer从sidecar拆分出来带来的架构优势就更加明显。

就如图中，英勇无敌的阿克琉斯，手持盾牌，就不惧箭雨。

![](images/achilles.jpg)

> 备注：后面会单独出一个系列文章，详细介绍mixer cache的工作机制，外加源码分析。在本文中我尽量简化以便聚焦我们的关注点。

## Mixer缓存的问题

有了这个知识作为背景，我们开始本文的正题：这个mixer缓存的问题在哪里？为什么我称之为**阿克琉斯之踵**？

我们将关注点看到这里：“a=1,b=2,c=3”。这是经过简化之后的实际被adapter使用的属性名和属性值的表示，表示这里有三个属性以及他们的当前值。

我们做一个简单的假设：如果a/b/c三个属性的取值范围都只有100个，那么，envoy中的mixer cache理论上最多有多少缓存项？

![](images/cache-account.jpg)

很明显，Mixer Cache的数量=属性a的取值数量 * 属性b的取值数量 * 属性c的取值数量 = 100 * 100 * 100 = 100万。从数学的角度说，这里是每个属性项取值范围的笛卡尔乘积。

## 讨论和反馈

TBD：等收集后整理更新

## 后记

有兴趣的朋友，请联系我的微信，加入Dream Mesh内部讨论群。