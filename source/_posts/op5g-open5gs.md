---
title: 开源5G核心网调研——Open5GS
date: 2023-08-16 17:25:21
tags:
- 5GC
photos:
- ../asset/op5gc/1676013310374-4c1301a7-6212-4243-91cc-7fabcb5be496.png
---

# 背景
## 目标
OSM+Kubernetes为应用运维工作带来了的诸多优势，我们计划提供一个全流程的应用部署运维示例以及详尽技术文档帮助全面掌握工具的使用方式。
下一阶段工作我们将基于此工作平台部署一个5G核心网应用实例，并实现应用指标监测、自动/手动缩扩容以及故障处理等主要功能。
0备选的开源5G核心网项目有：free5GC，Open Air Interface，open5GS。接下来将对这三个项目进行技术调研，最终选出用于部署工作的项目。
本篇文章将对[Open5GS](https://open5gs.org/open5gs/)项目进行简单调研。
## 项目背景
Open5gs与free5GC类似是由开源组织负责维护的5G核心网应用解决方案。Open5GS提供了5G 核心网和 EPC 的开源实现，即LTE/NR网络的核心网络标准实现（[Release-16](https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=3144)）。
![](../asset/op5gc/1676013310374-4c1301a7-6212-4243-91cc-7fabcb5be496.png#averageHue=%2347704c&clientId=u8000ddf1-eacf-4&from=markdown&height=128&id=CFcqj&originHeight=256&originWidth=1024&originalType=url&ratio=1.25&rotation=0&showTitle=false&status=done&style=none&taskId=u7a3c5e5a-5fc0-41ea-be8f-607af833f05&title=&width=512)
与许多开源项目一样，Open5GS也是从用爱发电开始的，在达到一定可用性后再靠项目本身从捐助者和赞助商那里获取支持。没有人能够确定这样的模式能够养活开源项目，但好在应该很少有人会在没有稳定收入的情况下将自己的全部精力投入到开源中。
# 概况
Open5GS同样支持了4G/5G混合的非独立组网核心网架构和5G独立组网的核心网架构。
下图是Open5GS的项目架构图，可以看到Open5gs的控制平面同时实现了SA部署和NSA部署，其中5G核心网部分的网络功能可以独立部署。SA组网是5G网络未来的趋势，但NSA在4G向5G的过渡进程中也是不可或缺的，Open5gs这种结构的好处是既保留了NSA组网的过渡性质也为将来的SA组网提前做了准备。
![](../asset/op5gc/1676015231610-66938d92-7aab-4573-8b0b-778cceb15a2f.png#averageHue=%23bfd590&clientId=u8000ddf1-eacf-4&from=markdown&id=AxsZ1&originHeight=725&originWidth=1010&originalType=url&ratio=1.25&rotation=0&showTitle=false&status=done&style=none&taskId=u850cb1bf-a3d6-47de-8a78-0c2bbb4b880&title=)
# 项目分析
借着分析Open5gs的SA模式与NSA模式顺便描述4G EPC向5G CN架构过渡中NFs的变迁。
## NSA
Open5gs的非独立组网实现了如下的网络功能组件：

- MME - Mobility Management Entity
- HSS - Home Subscriber Server
- PCRF - Policy and Charging Rules Function
- SGWC - Serving Gateway Control Plane
- SGWU - Serving Gateway User Plane
- PGWC/SMF - Packet Gateway Control Plane / (component contained in Open5GS SMF)
- PGWU/UPF - Packet Gateway User Plane / (component contained in Open5GS UPF)

MME是核心网络中主要的控制平面集线器（Hub），主要管理会话、移动性、寻呼和承载。MME链接到 HSS，HSS生成 SIM 认证向量并保存用户以及 SGWC 和 PGWC/SMF的配置文件。移动网络中的所有eNB（4G基站）都连接到MME。控制平面的最后一个元素是 PCRF，它位于 PGWC/SMF 和 HSS 之间，负责处理计费和执行用户策略。
用户平面在eNB/NSA gNB和外部WAN之间承载用户数据包（PDU）。SGWU和PGWU/UPF是用户平面的两个核心组件，它们会连接回它们在控制平面的对应物。eNB/NSA gNB 连接到 SGWU，SGWU 连接到 PGWU/UPF，然后再连接到 WAN。通过像这样物理分离控制平面和用户平面，大大增加了核心网络面对不同流量时的灵活性——这意味着可以针对不同的流量规模部署多个用户平面服务器（例如具有高速 Internet 连接的地方），同时保持控制功能集中。
## SA
Open5gs的独立组网实现了如下的网络功能组件：

- NRF - NF Repository Function
- SCP - Service Communication Proxy
- AMF - Access and Mobility Management Function
- SMF - Session Management Function
- UPF - User Plane Function
- AUSF - Authentication Server Function
- UDM - Unified Data Management
- UDR - Unified Data Repository
- PCF - Policy and Charging Function
- NSSF - Network Slice Selection Function
- BSF - Binding Support Function

5G SA核心网的工作方式与4G核心网不同——它使用基于服务的架构 (SBA)。控制平面NFs向NRF 注册，然后 NRF 帮助它们发现其他的核心网NF。AMF处理连接和移动性管理，其功能是4G MME 任务的一个子集。gNB（5G 基站）连接到 AMF，UDM、AUSF和UDR执行与4G HSS类似的操作，生成SIM认证向量并保存用户配置文件。会话管理全部由SMF处理（以前由 4G MME/ SGWC/ PGWC 负责）。NSSF 提供了一种选择网络切片的方法，PCF 用于计费和执行用户策略。
5G SA 核心网的用户平面要简单得多，因为它只包含了一个功能。UPF在gNB和外部 WAN 之间承载用户数据包。除SMF和UPF外，5G SA核心功能的所有配置文件仅包含功能的IP绑定地址/本地接口名称和NRF的IP地址/DNS 名称。
# 活跃度
为了追求极致的速度，Open5gs同样基于C语言编写
![](../asset/op5gc/1676021617466-37716d86-b40d-473f-9d9e-0bc2c8e24570.png#averageHue=%23f4f3f2&clientId=u8000ddf1-eacf-4&from=markdown&id=Q4fWm&originHeight=162&originWidth=405&originalType=url&ratio=1.25&rotation=0&showTitle=false&status=done&style=none&taskId=u754f4851-d7c7-4cc3-88f9-19cf752bd35&title=)
Open5gs代码仓库在Github上已经获得了1.1k的star
![](../asset/op5gc/1676021607015-6ba24891-3cf6-4e06-82c3-19031374ffc7.png#averageHue=%23eef1f3&clientId=u8000ddf1-eacf-4&from=markdown&id=UsDbW&originHeight=54&originWidth=695&originalType=url&ratio=1.25&rotation=0&showTitle=false&status=done&style=none&taskId=u054a20b0-d999-450b-9e2c-ab1643ebc52&title=)
截至目前已有60+开发者对此项目贡献代码
![](../asset/op5gc/1676021634201-763e37e7-9793-499d-8067-282904bc4283.png#averageHue=%23f7f6f5&clientId=u8000ddf1-eacf-4&from=markdown&id=u6jdn&originHeight=205&originWidth=379&originalType=url&ratio=1.25&rotation=0&showTitle=false&status=done&style=none&taskId=ud4a01161-2f5a-49e9-93ef-a0829468f98&title=)
Open5gs的讨论区也比较活跃，几乎每天都会有新的Issues提交，其中包括Bug反馈、需求发布以及问题求助等内容。
![](../asset/op5gc/1676021641823-cba74528-aec8-4c57-b9f5-bdb88786261c.png#averageHue=%23fefdfc&clientId=u8000ddf1-eacf-4&from=markdown&id=Ki30J&originHeight=869&originWidth=1532&originalType=url&ratio=1.25&rotation=0&showTitle=false&status=done&style=none&taskId=uf7dafdb6-3b80-4333-ae61-00141875bd0&title=)
# 可用性及限制
Open5gs官网提供了step by step的安装教程，针对各种环境的差异提供了不同步的安装策略供参考。
![](../asset/op5gc/1676022198251-a390c96c-e468-4e20-a0e0-6c655175d045.png#averageHue=%23fcfcfc&clientId=u8000ddf1-eacf-4&from=markdown&height=280&id=a303k&originHeight=373&originWidth=629&originalType=url&ratio=1.25&rotation=0&showTitle=false&status=done&style=none&taskId=u72e1ac2b-93d7-421a-8c3c-e36c8ef9c34&title=&width=472)
不止于此，Open5gs还提供了基于[Prometheus](https://prometheus.io/docs/introduction/overview/)的指标监控集成，省去了单独部署指标监控组件以及自定义指标的麻烦。
![](../asset/op5gc/1676022467037-fff64a6f-0227-47f9-812c-7dfb5ba21100.png#averageHue=%23fcfbfa&clientId=u8000ddf1-eacf-4&from=markdown&id=Wb8im&originHeight=537&originWidth=1459&originalType=url&ratio=1.25&rotation=0&showTitle=false&status=done&style=none&taskId=ud97313cf-dcae-40a4-a24d-992b89f158f&title=)
在Github上分别搜索Open5gs和free5GS，可以看到两者的各项数据都比较接近。二者最大的区别是前者基于C语言，后者基于Golang。
![](../asset/op5gc/1676023338827-15f66fc7-4183-4e4c-89ec-b91853dd66c7.png#averageHue=%23fefdfd&clientId=u8000ddf1-eacf-4&from=markdown&height=373&id=flQDp&originHeight=497&originWidth=621&originalType=url&ratio=1.25&rotation=0&showTitle=false&status=done&style=none&taskId=u47af1c24-13cf-455d-bd86-93ad5567702&title=&width=466)

# 总结
作为与free5GC类似规模的开源项目，Open5gs提供了更为全面的使用教程，但一部分文档发布时间较早缺乏维护，很可能随着版本更新其中的内容以不再准确
