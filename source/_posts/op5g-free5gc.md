---
title: 开源5G核心网调研——free5GC
date: 2023-08-15 10:18:12
tags:
- 5GC
photos:
- ../asset/op5gc/free5gclogo.png
---

# 背景
## 目标
OSM+Kubernetes为应用运维工作带来了的诸多优势，我们计划提供一个全流程的应用部署运维示例以及详尽技术文档帮助全面掌握工具的使用方式。
下一阶段工作我们将基于此工作平台部署一个5G核心网应用实例，并实现应用指标监测、自动/手动缩扩容以及故障处理等主要功能。
备选的开源5G核心网项目有：free5GC，Open Air Interface，open5GS。接下来将对这三个项目进行技术调研，最终选出用于部署工作的项目。
## 项目背景
:::info

- free5GC 是第 5 代 (5G) 移动核心网络的开源项目。 该项目的最终目标是实施 3GPP 第 15 版 (R15) 及更高版本中定义的 5G 核心网 (5GC)。
:::
[Free5GC](https://www.free5gc.org/)是台湾同胞维护的5G核心网开源项目，目前由台湾国立交通大学 （NCTU）在维护。
## 5G核心网系统架构主要特征
将5G核心网与4G核心网EPC进行比较，可以看出5G相比4G在基本功能如认证、移动性管理、连接、路由等方面不变，但是方式和技术手段发生了变化，更加灵活。主要体现在：移动性管理（AMF）和会话管理（SMF）分离，AMF和SMF的部署可层级分开；承载与控制分离，UPF和SMF的部署层级也可以分开；AMF和UPF根据业务需求、信令和话务流量以及传输资源灵活部署；采用服务化架构设计，网元功能进行了模块化解耦，接口进行了简化。总体上看，5C 核心网的组网更加灵活，但部署灵活性也对传输、以及网络规划、网络运营管理等能力提出更高的要求。
5G核心网架构为用户提供数据连接和数据业务服务，基于NFV（Network Function Virtualization，网络功能虚拟化）和 SDN（Software Defined Network，软件定义网络）等新技术，其控制面网元之间使用服务化的接口进行交互。5G核心网系统架构主要特征如下：

1. 承载和控制分离：承载和控制可独立扩展和演进，可集中式或分布式灵活部署；
2. 模块化功能设计：可以灵活和高效地进行网络切片；
3. 网元交互流程服务化：按需调用，并服务可重复使用；
4. 每个网元可以与其他网元直接交互，也可通过中间网元辅助进行控制面的消息路由；
5. 无线接入和核心网之间弱关联：5G核心网是与接入无关并起到收敛作用的架构，3GPP和非3GPP均通过通用的接口接入5G核心网；
6. 支持统一的鉴权框架；
7. 支持无状态的网络功能，即计算资源与存储资源解耦部署；
8. 基于流的QoS：简化了QoS架构，提升了网络处理能力； 9) 支持本地集中部署的业务的大量并发接入， 用户面功能可部署在靠近接入网络的位置，以支持低时延业务、本地业务网络接入。
:::info

- NFV技术通过软件与硬件的分离，为5G网络提 供更具弹性的基础设施平台，组件化的网络功 能模块实现控制面功能可重构。NFV使网元功 能与物理实体解耦，采用通用硬件取代专用硬 件，可以方便快捷的把网元功能部署在网络中 任意位置，同时对通用硬件资源实现按需分配 和动态伸缩，以达到最优的资源利用率。
- SDN 技术实现控制功能和转发功能的分离。控制功 能的抽离和聚合，有利于通过网络控制平面从 全局视角来感知和调度网络资源，实现网络连 接的可编程。
:::
# 概况
【一句话结论】
一开始的fress5GC项目只是将4G EPC基于5G核心网的SBA（Service Based Architecture）架构做了迁移，stage-1版本基于nextEPC（4G EPC R13 的一个实现）项目实现。其中将MME、SGW、PGW迁移到5GC，策略和计费规则功能 (PCRF) 和归属用户服务器 (HSS) 保持不变。
> SBA架构借鉴IT系统微服务的理念，通过模块化实现网络功能间的解耦和整合，各解耦后的网络功能（服务）可以独立扩容、独立演进、按需部署；各种服务采用服务注册、发现机制，实现了各自网络功能在5G核心网中的即插即用、自动化组网；同一服务可以被多种NF调用，提升服务的重用性，简化业务流程设计。SBA设计的目标是以软件服务重构核心网，实现核心网软件化、灵活化、开放化和智慧化。

Stage 1的参考架构如下图所示，这并不是严格意义上符合3GPP标准的核心网架构，只是基于微服务的思想将网络功能做了拆分。
![Fig. 1: Stage 1 architecture of free5GC](../asset/op5gc/b632f0bcedccea6b.png#height=314&id=OYysk&originHeight=419&originWidth=1035&originalType=binary&ratio=1&rotation=0&showTitle=true&status=done&style=shadow&title=Fig.%201%3A%20Stage%201%20architecture%20of%20free5GC&width=776 "Fig. 1: Stage 1 architecture of free5GC")
2019年发布的stage-2版本符合[3GPP](https://www.3gpp.org/release-16)标准。
![Fig. 2: Stage 2 architecture of free5GC](../asset/op5gc/1abcd46b7c03b9f7.png#height=375&id=l0ckk&originHeight=500&originWidth=1072&originalType=binary&ratio=1&rotation=0&showTitle=true&status=done&style=shadow&title=Fig.%202%3A%20Stage%202%20architecture%20of%20free5GC&width=804 "Fig. 2: Stage 2 architecture of free5GC")

# 项目分析
free5GC stage 1的最新提交停留在了2019年6月份，看得出来开发者已经停止维护了，目前主要维护的应该是stage 3版本。本身stage 1也并不是一个符合标准的核心网应用，故也不再我们的选型范围之内。
![Fig. 3:  Source Code of free5GC Stage 1](../asset/op5gc/8b4a8e37286afe3c.png#height=710&id=gcV3M&originHeight=946&originWidth=1266&originalType=binary&ratio=1&rotation=0&showTitle=true&status=done&style=shadow&title=Fig.%203%3A%C2%A0%20Source%20Code%20of%20free5GC%20Stage%201&width=950 "Fig. 3:  Source Code of free5GC Stage 1")
stage 2与stage 1情况类似，在2020年停止了开发，此处略过。
stage 3目前还处于持续开发中，最新的Release版本是半年前发布的V3.2.1。
![Fig. 4: free5GC Stage 3 Release v3.2.1](../asset/op5gc/85915034e1dd7fa9.png#id=zjLCy&originHeight=773&originWidth=1439&originalType=binary&ratio=1&rotation=0&showTitle=true&status=done&style=shadow&title=Fig.%204%3A%20free5GC%20Stage%203%20Release%20v3.2.1 "Fig. 4: free5GC Stage 3 Release v3.2.1")
控制面数据面分离
通信协议
# 活跃度
目前该项目的[Github仓库](https://github.com/free5gc/free5gc)共有1.5k star，有29个人贡献代码，2023年也已经有数个commit，可以看出项目还在持续维护。
![Fig. 5: Star](../asset/op5gc/1675771631560-a796d799-deb8-4266-a6e5-1875157d07bf.png#averageHue=%23eef1f4&clientId=u1ceae4c1-3419-4&from=markdown&height=42&id=mHicm&originHeight=56&originWidth=586&originalType=url&ratio=1&rotation=0&showTitle=true&status=done&style=shadow&taskId=u69457789-1637-4e7f-aa2c-9210eb2f15f&title=Fig.%205%3A%20Star&width=440 "Fig. 5: Star")
![Fig. 6: Contributors](../asset/op5gc/f27c35eb9e5892ab.png#height=158&id=nZNGo&originHeight=210&originWidth=376&originalType=binary&ratio=1&rotation=0&showTitle=true&status=done&style=shadow&title=Fig.%206%3A%20Contributors&width=282 "Fig. 6: Contributors")
项目官方维护了一个[论坛](https://forum.free5gc.org/)供使用者交流经验/反馈BUG，但论坛并不是十分活跃，平均2-3天一个帖子，以求助类型帖子为主，且并不是每个帖子都有人回复。
![Fig. 5: Forum](../asset/op5gc/eedd0e904fa967b7.png#height=294&id=kvaQC&originHeight=588&originWidth=715&originalType=binary&ratio=1&rotation=0&showTitle=true&status=done&style=shadow&title=Fig.%205%3A%20Forum&width=358 "Fig. 5: Forum")
# 可用性及限制
项目官方提供了关于安装测试的文档[free5GC Documentation](https://github.com/free5gc/free5gc/wiki)，由于是台湾大学维护的项目所以部分文档也有[繁体中文版本](https://www.free5gc.org/installations/stage-3/)。
![](../asset/op5gc/1675773309074-2a3cb782-32eb-4e19-9d28-5254c7444fc9.png#averageHue=%23fefefe&clientId=u1ceae4c1-3419-4&from=markdown&id=cy0TU&originHeight=1019&originWidth=1584&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=shadow&taskId=u109b47ac-6778-4a16-a830-14325cde7e7&title=)
官方文档的教程基于Linux裸机环境，并没有Kubernetes部署的教程。但在云原生席卷一切的趋势下也有很多技术大佬发布了在Kubernetes上部署free5GC的资料，比如[towards5gs-helm](https://github.com/Orange-OpenSource/towards5gs-helm), [free5gc-k8s](https://github.com/sumichaaan/free5gc-k8s)以及[free5GC-stage3-on-k8s](https://github.com/chih-hsi-chen/free5GC-stage3-on-k8s)。具体有效性尚待验证。这些项目多直接使用K8s的yaml文件直接创建Service或者使用Helm免去了手动部署众多服务的麻烦，但在实际使用和运维的过程中的便利性，如果最终确定使用此项目那么我们将会开发一个基于juju的集成了检测、运维、缩扩容功能的应用版本。
仔细研究了[towards5gs-helm](https://github.com/Orange-OpenSource/towards5gs-helm)这个项目，此项目使用helm chart进行核心网络的部署，而helm也是OSM支持的与juju类似的工具，若选用free5GC则可以参考此项目使用helm进行工作。
# 总结
总的来说本项目麻雀虽小五脏俱全，社区文档应有尽有，但生态并不是十分活跃，后期如果出现较复杂的BUG或者困惑可能很难寻求到帮助。
