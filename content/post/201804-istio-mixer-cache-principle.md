+++
title = "Istio Mixer Cache工作原理与源码分析(2)－工作原理"

date = 2018-04-29
lastmod = 2018-04-29
draft = true

tags = ["Istio", "Mixer"]
summary = "Mixer Cache设计时牵绊很多，非常的不好实现，现有的这套方案，很有些束手束脚的感觉。我们先在本文中概括讲述mixer cache的工作原理，部分细节留给后面展开。"
abstract = "Mixer Cache设计时牵绊很多，非常的不好实现，现有的这套方案，很有些束手束脚的感觉。我们先在本文中概括讲述mixer cache的工作原理，部分细节留给后面展开。"

[header]
image = "headers/dreammesh-architecture-2.jpg"
caption = ""

+++

## 前言

经过前面的基础概念的介绍，我们现在已经可以勾勒出一个mixer cache的实现轮廓，当然实际代码实现时会有很多细节。但是为了方便理解，我们在深入细节之前，先给出一个简化版本，让大家快速了解mixer cache的实现原理。后面的章节我们再逐渐深入。

## Mixer Cache的构造

Mixer Cache在实现时，在envoy的内存中，保存有两个数据结构：

```c++
class CheckCache {
  std::unordered_map<std::string, Referenced> referenced_map_;
  using CheckLRUCache = utils::SimpleLRUCache<std::string, CacheElem>;

  std::unique_ptr<CheckLRUCache> cache_;
}
```

> 具体代码: 见istio/proxy项目，文件`src/istio/mixerclient/check_cache.h`

- referenced_map

  referenced_map保存的是引用属性，key是这些引用属性的签名。

  签名的具体算法我们稍后深入，这里理解成某种特殊的hash算法即可。

- cache

  cache保存的是操作结果，封装成CacheItem对象，key同样是引用属性的签名。

## 保存Check结果到Mixer Cache

当第一次访问或者没有匹配到缓存时（后面再谈如何匹配缓存），这时需要访问mixer，得到处理结果和随response而来的referencedAttributes。

envoy保存mixer结果到缓存的主要步骤是：

1. ​