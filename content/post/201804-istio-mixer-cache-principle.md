+++
title = "Istio Mixer Cache工作原理与源码分析(2)－工作原理"

date = 2018-06-05
lastmod = 2018-06-05
draft = true

tags = ["Istio", "Mixer"]
summary = "Mixer Cache设计时牵绊很多，非常的不好实现，现有的这套方案，很有些束手束脚的感觉。我们先在本文中概括讲述mixer check cache的工作原理，部分细节留给后面展开。"
abstract = "Mixer Cache设计时牵绊很多，非常的不好实现，现有的这套方案，很有些束手束脚的感觉。我们先在本文中概括讲述mixer check cache的工作原理，部分细节留给后面展开。"

[header]
image = "headers/dreammesh-architecture-2.jpg"
caption = ""

+++

## 前言

经过前面的基础概念的介绍，我们现在已经可以勾勒出一个mixer cache的实现轮廓，当然实际代码实现时会有很多细节。但是为了方便理解，我们在深入细节之前，先给出一个简化版本，让大家快速了解mixer cache的实现原理。后面的章节我们再逐渐深入。

Mixer Cache分为两个部分：

1. check cache
2. quota cache

简单起见，我们先关注check cache，在check cache讲述清楚之后，我们再继续看quota cache。

>  备注：istio一直在持续更新，以下代码来源于istio 0.8版本。

## Mixer Check Cache的构造

Mixer Cache在实现时，在envoy的内存中，保存有两个数据结构：

```c++
class CheckCache {
  std::unordered_map<std::string, Referenced> referenced_map_;
    
  using CheckLRUCache = utils::SimpleLRUCache<std::string, CacheElem>;
  std::unique_ptr<CheckLRUCache> cache_;
}
```

> 具体代码: 见istio/proxy项目，文件`src/istio/mixerclient/check_cache.h`

1. referenced_map

   referenced_map保存的是引用属性，key是这些引用属性的签名。

   签名的具体算法我们稍后深入，这里理解成某种特殊的hash算法即可。

1. cache

   cache保存的是操作结果，封装成CacheItem对象，key同样是引用属性的签名。


这里和一般缓存不一样，有两个map，也就是存在两套key/value，为什么要这样设计？

## Mixer Check Cache的核心设计

cache在设计上，最核心的内容就是如何设计cache的key，这个问题在mixer check cache中尤其突出。

### 为什么要有两层Map？

我们继续以这个最基本的场景为例：

![](../201804-istio-mixer-cache-concepts/images/referenced-attributes.jpg)

注意这个场景下属性的使用情况是这样的：

- envoy提交的请求中有5个属性，”a=1,b=2,c=3,e=0,f=0”
- mixer中有三个adapter，每个adapter只使用提交属性中的一个属性a/b/c
- 在CheckResponse中返回referencedAttributes字段的内容为”a,b,c”

要怎么设计这个Mixer check cache？先分析缓存的逻辑语义：

1. 返回的referencedAttributes字段的内容为”a,b,c”，说明这三个属性被使用
2. 结合输入的”a=1,b=2,c=3,e=0,f=0”，就可以得知"a=1,b=2,c=3"这个属性和属性的值的组合，代表一个输入，结果是固定而可以缓存的
3. 如果下一个请求，同样提供”a,b,c”三个属性，并且三个属性的值是"a=1,b=2,c=3"，则可以直接使用这个缓存的结果

注意：由于哪些属性可能会被使用是取决于运行时实际部署的adapter，因此mixer check cache的key计算时是无法直接指定要计算哪些属性的，也就无法简单的对输入属性做简单计算得到key。这是mixer cache和一般场景下的缓存的关键差异。

mixer check cache在工作时，如果要命中缓存，就必须带有两层匹配逻辑：

1. 请求中是否携带有匹配的属性，在上面的例子中，就是要有”a,b,c”三个属性
2. 这些属性是否具备匹配的值，在上面的例子中，就是要”a=1,b=2,c=3”

在具体实现上：

- referenced_map 是第一层缓存，用来保存被缓存的属性组合
- cache 是第二层缓存，用来保存输入的签名（根据引用属性计算而来）/value （check的检查结果）

### 两层cache是如何工作的？

为了避免陷入代码细节，我们先不看代码具体实现（这是下一章的内容），先只看工作原理：

- referenced_map 用来保存哪些属性组合已经被缓存，比如 `{"k1": "a,b,c"}` 这样表示当前只有一个属性组合"a,b,c"被保存，为了简单我们先忽略这里key的计算方式。
- cache用来保存输入的签名(简单理解为有效输入内容”a=1,b=2,c=3”的hash结果)和check 结果（简化为true/false表示是否通过），比如 `{ "a=1,b=2,c=3": "true" }`

我们来看各种场景下的请求和缓存的匹配请求，先看最理想的缓存命中的场景：

- 请求为：”a=1,b=2,c=3,e=0,f=0”

    这个请求和被缓存的请求是一模一样的，我们期待可以命中缓存。

    匹配时，先进行第一层匹配：输入的”a=1,b=2,c=3,e=0,f=0”和 referenced_map {"k1": "a,b,c"} 进行检查，发现输入的”a=1,b=2,c=3,e=0,f=0”可以和保存的"a,b,c"属性组合匹配。

    然后继续，第二层缓存就可以简单通过key来匹配了。注意在对输入进行签名时，只需要计算引用属性的hash值，即只需要计算"a=1,b=2,c=3"，再通过这个签名在cache中找到缓存结果。

    这便是标准的mixer check cache的匹配姿势。

- 请求为：”a=1,b=2,c=3,e=1,f=2”

    差异在于e/f属性的值有所不同，考虑到e/f两个属性没有adapter使用，和”a=1,b=2,c=3,e=0,f=0”等效，我们期待可以命中缓存。

    第一层匹配，输入的”a=1,b=2,c=3,e=1,f=2”和{"k1": "a,b,c"} 命中，由于属性组合是"a,b,c"，因此计算签名时还是计算"a=1,b=2,c=3"，因此可以命中第二层缓存。

    通过这种在签名时忽略未被adapter使用的属性的方式，mixer check cache 做到了只检查被adapter使用的属性，而其他属性的值不会影响。

我们再来看缓存不命中的典型场景，此时会多一个保存新结果到缓存的过程：

- 新请求：”a=1,b=2,c=10,e=0,f=0”

    不同在于c的取值有变化，这是一个新的有效输入，和已经缓存的"a=1,b=2,c=3"不同，应该无法命中。

    匹配时，第一层匹配命中，计算签名时计算的输入是"a=1,b=2,c=10"，得到的签名结果自然和缓存的"a=1,b=2,c=3"的签名不同，因此第二层缓存没有命中。

    这是典型的属性组合匹配但是属性具体值不匹配的场景，我们看mixer check cache的后续处理。

    缓存不命中，就需要向mixer发起远程，得到应答，应答中给出adapter使用的属性情况，此时依然是"a,b,c"，和检查的结果，我们假定这次是false。即此时我们得到了一个新的输入和结果的对应关系，我们将这个结果保存起来：referenced_map 中现有的值是 {"k1": "a,b,c"}，无需改变。cache 从 { "a=1,b=2,c=3": "true" } 增加新结果，变为  { "a=1,b=2,c=3": "true", "a=1,b=2,c=10": "false"}

- 继续发送请求：”a=1,b=2,c=10,e=0,f=0”/”a=1,b=2,c=3,e=0,f=0”

	如果继续有这两个请求进来，则继续命中。

- 新请求：”a=1,b=20,c=10,e=0,f=0”

	如果属性a/b/c的值继续变化，则继续重复前面的不命中后更新缓存的步骤。

### absent key

通过上面稍显枯燥的描述，我想大家基本可以了解 mixer check cache 的工作原理，但是注意这个是经过很多简化的最初级版本，我们现在来稍微复杂一点，加上 absent key 的概念。

什么叫做 absent key ？我们需要继续看回这个图片，注意mixer adapter使用的属性是a/b/c三个：

![](../201804-istio-mixer-cache-concepts/images/referenced-attributes.jpg)

前面我们列出来的所有场景中，每个输入中都包含有a/b/c三个属性，考虑到其他不使用的属性在匹配过程中会被忽略而不影响，我们来将关注点放在a/b/c三个属性上。需要考虑一种可能：如果a/b/c三个属性不是每次都同时提供，而是少一个或者多个，结果会怎么样？

此时两层缓存的数据为：

- referenced_map = {"k1": "a,b,c"}
- cache = { "a=1,b=2,c=3": "true", "a=1,b=2,c=10": "false"}


如果我们有一个输入 ”a=1,b=2,c不存在,e=0,f=0” ，这个输入中 c没有出现的。如果我们不做调整，继续沿用上面的缓存逻辑，那么处理情况会是如此：

1. 第一层缓存 referenced_map = {"k1": "a,b,c"} 和输入”a=1,b=2,c不存在,e=0,f=0” 因为c的缺席而无法匹配

2. 只能发起对mixer的请求，获取新的应答，假设这种情况下由于c不存在，mixer的adapter有不一样的处理结果，返回false。

注意此时两个输入和结果："a=1,b=2,c=3"结果为"true"，"a=1,b=2,c不存在"结果为"false"。对于mixer adapter的response，在返回引用属性时，就有两种选择：

1. 只返回a/b
2. 返回a/b，但是同时指出

3. a/b/c属性（注意c属性是被adapter实际使用了的，包括不存在也是某种使用），结果为false

4. 更新缓存，第一层缓存更新为 referenced_map = {"k1": "a,b,c", "k2": "a,b" }，第二层缓存更新为 cache = { "a=1,b=2,c=3": "true", "a=1,b=2,c=10": "false", "a=1,b=2": "false"}