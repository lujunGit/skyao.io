+++
title = "DreamMesh抛砖引玉(10)-多集群"

date = 2018-03-23
lastmod = 2018-03-23
draft = true

tags = ["DreamMesh"]
summary = "如果有多集群/多机房的支持需求，该如何解决？这个问题和前面列出的service mesh体系和非service mesh的并存问题，可能叠加：如何在多集群/多机房要求下实现service mesh体系和非service mesh的并存。"
abstract = "如果有多集群/多机房的支持需求，该如何解决？这个问题和前面列出的service mesh体系和非service mesh的并存问题，可能叠加：如何在多集群/多机房要求下实现service mesh体系和非service mesh的并存。"

[header]
image = "headers/dreammesh-brainstorm-10.jpg"
caption = ""

+++

如果有多集群/多机房的支持需求，该如何解决？

这个问题和前面列出的service mesh体系和非service mesh的并存问题，可能叠加：如何在多集群/多机房要求下实现service mesh体系和非service mesh的并存。

## 背景

首先看看需求来自哪里：

1. Service Mesh体系和非Service Mesh体系并存/过渡的需要

	见前文 [DreamMesh抛砖引玉(3)-艰难的过渡](../201802-dreammesh-brainstorm-transition/)

2. 多机房部署

	- 有些是考虑容灾，异地双活以备不时之需
	- 大多数没有这么高要求，只是地域差异，应用部署在不同地域
	- 有些是部门/组织/系统不同，各自部署独立系统

3. 技术栈差异

	不同时期使用的技术栈不同，可能造成有多套集群，比如原有dubbo，现在要转向spring cloud。这样在过渡期间会有两套系统。

	如果公司较大，也会出现不同组织采用不同的技术栈导致出现多个集群。

4. 测试运维需要

	可能希望在生产环境之外搭建stagging，test等集群，通常是独立运作，但是偶尔会有想法，希望开个口子以便打通若干个服务或者某个实例，来做测试和验证。

## 分析

我们要打通的多个微服务集群，情况很复杂，可能是以下多种场景混杂：

1. 集群可能是Service Mesh体系和非Service Mesh体系

	- 如果是Service Mesh，可能是Istio/Conduit
	- 如果是非Service Mesh体系，可能是SpringCloud/Dubbo/Motan等

1. 集群可能有不同的部署方式

	- 不同地域
    - 不同机房
    - 网络不同，可能在虚拟网络中如k8s

1. 集群可能使用不同的技术栈，包括：

	- 服务注册机制
	- 远程通讯机制

1. 集群可能用于不同的产品部署阶段

	- prod
	- stagging
	- test



## 讨论和反馈

TBD：等收集后整理更新

## 后记

有兴趣的朋友，请联系我的微信，加入DreamMesh内部讨论群。
