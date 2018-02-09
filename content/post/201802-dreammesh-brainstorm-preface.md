+++
title = "DreamMesh抛砖引玉之序言"

date = 2018-02-09
lastmod = 2018-02-09
draft = false

tags = ["DreamMesh"]
summary = "近期在Service Mesh如何落地上，有些想法和思路，真诚的邀请朋友们参与讨论，希望能引出更多更好的内容。"
abstract = "近期在Service Mesh如何落地上，有些想法和思路，真诚的邀请朋友们参与讨论，希望能引出更多更好的内容。"

[header]
image = "headers/dreammesh-brainstorm-1.jpg"
caption = "重型猎鹰发射升空"

+++

## 前言

相信能看到这篇博客的同学，大体都知道过去几个月间，我在努力的研究Service Mesh并致力于在国内拓荒和布道。

坦言说：Service Mesh的发展进程，当前还处于前景虽然一致看好，但是脚下的路还是需要一步一步走的早期艰难阶段。由于istio和conduit的尚未成熟，造成目前Service Mesh青黄不接的尴尬局面。

近期在和很多朋友交流，也都谈到这个话题，着实有些无奈，只能静静的等待istio和conduit的发展。好在这两个产品也很争气，近期都快速发出了新版本。

然而Service Mesh的现状，它的不够成熟，终究还是引发了猜疑，不安和观望。

![](images/doubt-kills-dreams.jpg)

**Doubt kills more dreams than failure ever has**

在猎鹰升空，马斯克封神的本周，我更能深刻体会这句话的内涵。

## 正视问题

过去的一个月间，我一直在认真的思考这样一个问题：

**Service Mesh真的能完美解决问题吗？**

这里我们抛开前面所说的istio或者conduit的暂未成熟，毕竟这是在不远的未来即将看到（至少有希望）能被解决，不出意外2018年年中或者年底，这个成熟度还是有望在很大程度上解决。

让我们设想：如果明天Istio或者conduit发布出Production Ready的版本，是不是我们就可以欢欣鼓舞的直接将系统搬迁到service mesh上？

**还差点什么？**

我将会在稍后的系列文章中将我思考的问题逐个列出来，暂时会有下列内容：

- Ready for Cloud Native?

    我对Service Mesh的第一个担忧，来自 **Cloud Native**。

    作为Cloud Native的忠实拥护者，我不怀疑Cloud Native的价值和前景。但是，我担心的是：准备上service mesh的各位，是否都已经做到了ready for Cloud Native？

- 如何从非Service Mesh体系过渡到Service Mesh？

	即使一切都ready，对于一个有存量应用的系统而言，绝无可能在一夜之间就将所有应用都改为Service Mesh，然后一起上线。

	必然会有一个中间过渡状态，一部分应用开始搬迁到Service Mesh，大部分还停留在原有体系。那么，如果在过渡阶段让Service Mesh内的服务和Service Mesh外的服务相互通讯？

- 零侵入的代价

	Service Mesh的一个突出有点就是对应用的侵入度非常低，低到可以"零侵入"。

    然而，没有任何事情是可以十全十美的，零侵入带来的弊端：iptables一刀切的方案有滥用嫌疑，为了不劫持某些网络访问又不得不增加配置去绕开。

	是否考虑考虑补充一个低侵入的方案？

- 网络通讯方案

	Service Mesh解决服务间通讯的方式是基于网络层，HTTP1.1/HTTP2和可能的TCP，对于选择什么样的序列化方案和远程访问方案并未限制。

	好处是我们可以自由的选择各种方案，坏处就是自由意味着自己做。

    能否以类库或者最佳实践的方式给出适合service mesh时代的网络通讯方案？

- 绕不开的spring

	对于Java企业应用，spring是无论如何绕不开的。然而目前我们没有看到spring社区对service mesh的任何回应。因此，如何以正确的姿势在service mesh时代使用spring，需要自己探索。

	理论上说，springboot on service mesh有可能成为一个清爽的解决方案。然后路终究是要走一遍才知道是不是那么美好。

- spring cloud

	虽然service mesh号称下一代微服务，取代spring cloud是service mesh与生俱来的天然目标之一。

	但是以目前市场形式，spring cloud在未来很长一段时间之类都会是市场主流和很多公司的第一选择。如何在搬迁到service mesh之前加强spring cloud，并为将来转入spring cloud铺路，是一个艰难而极具价值的话题。

- API gateway

	service mesh解决的是东西向服务间通讯的问题，我们来审视一下API gateway到微服务之间的南北向通讯： 服务发现，负载均衡，路由，灰度，安全，认证，加密，限流，熔断......几乎大部分的主要功能，在这两个方向上都高度重叠。

	因此，是否该考虑提供一个统一的解决方案来处理？

- 多集群/多机房的支持

	如果有多集群/多机房的支持需求，该如何解决？

	这个问题和前面列出的service mesh体系和非service mesh的并存问题，可能叠加：如何在多集群/多机房要求下实现service mesh体系和非service mesh的并存。

> TBD：更多想法将稍后逐步列出，也欢迎补充，请直接微信联系我。

## Dream Mesh

在经过一个月的冥思苦想和深度思考之后，我对上面列出的问题大致有了一些初步的想法和思路。

我个人心中理想的Service Mesh解决方案，希望是可以在istio或者conduit的基础上，进一步的扩展和完善，解决或者规避上述问题。

终极目标：让Service Mesh能够更加平稳的在普通企业落地。

这个美好的愿景，目前还只停留在我的脑海中，如梦境一般虚幻，又如梦境一般令人向往。

所以我将这个思路和解决方案，统称为"**Dream Mesh**"。

坦言说：Dream Mesh想法比较超前，规划也有点庞大，兼具高层架构和底层实现，极富挑战。

![](images/doubt-kills-dreams.jpg)

再一次用这张图片为自己打气，同时感谢太平洋对岸的埃隆·马斯克在本周这个关键的时间点上给了我更多的勇气。

## 诚邀

在将Dream Mesh的规划和架构正式呈现出来之前，在春节期间，我会陆续将我目前的所思所想以文字的形式发表在我的博客上，然后年后会发起几轮内部讨论。之后修订/补充/完善，希望在四五月份的时候能给出一个成型的方案。

我真诚的邀请对此感兴趣的朋友参与讨论和交流，我会在近期陆续将我的想法和设想抛出来，希望能引出大家的更多更好的思路，正所谓：抛砖引玉。

**没有什么事情是可以一个人闭门造车而独自琢磨出来的。**

当然乔布斯和张小龙不在此列，他们已经是一条腿迈进神的领域。而我等凡夫俗子，则更需要集体的智慧和力量。

我想能看到本文的同学，应该都是有我的联系方式的，请直接在微信上联系我加入内部讨论群。

十分期待。
