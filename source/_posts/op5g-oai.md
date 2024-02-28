---
title: 开源5G核心网调研——OpenAirInterface
date: 2023-08-15 15:07:27
tags:
- 5GC
photos:
- ../asset/op5gc/Snipaste_2024-02-28_15-13-27.png
---

# 背景
## 目标
OSM+Kubernetes为应用运维工作带来了的诸多优势，我们计划提供一个全流程的应用部署运维示例以及详尽技术文档帮助全面掌握工具的使用方式。
下一阶段工作我们将基于此工作平台部署一个5G核心网应用实例，并实现应用指标监测、自动/手动缩扩容以及故障处理等主要功能。
备选的开源5G核心网项目有：free5GC，Open Air Interface，open5GS。接下来将对这三个项目进行技术调研，最终选出用于部署工作的项目。
## 项目背景
OAI（OpenAirInterface）是2014年成立的法国非营利性组织，致力于5G无线网络技术的研究。OAI拥有活跃的开发人员社区，他们共同构建无线蜂窝无线电接入网络 (RAN) 和核心网络 (CN) 技术。
目前OAI开源的成熟项目包括[5G RAN](https://openairinterface.org/oai-5g-ran-project/)，[5G CN](https://openairinterface.org/oai-5g-core-network-project/)，[CI / CD](https://openairinterface.org/test-measurement/)以及[MASAIC5G](https://openairinterface.org/mosaic5g/)（一个旨在将无线接入 (RAN) 和核心网络 (CN) 转变为敏捷开放的网络服务交付平台）。本篇文章将对[5G CN](https://openairinterface.org/oai-5g-core-network-project/)项目进行调研。
# 概况
[OAI-5G 核心网项目](https://openairinterface.org/oai-5g-core-network-project/)（以下简称OAI）旨在提供具有丰富功能集的符合 3GPP 标准的 5G 独立 (Standalone, SA) CN 实施。OAI以灵活的方式设计和实施，可以轻松适应多样化的 5G 用例需求。OAI组件的所有功能都经过专业测试仪、商用 gNB和开源 RAN 模拟器的持续测试。
OAI也采用了面向服务的SBA架构，在此架构中核心网络的各个组件被划分为网络功能（NF），为其他授权的NF提供服务或访问其服务。
:::info

- 对于NFs之间的交互，其中一个充当服务消费者，另一个充当服务生产者。
- 控制平面 (CP) 功能与用户平面 (UP) 是分开的，以使它们能够独立扩展，从而使运营商可以使用这些组件来轻松地确定网络规模、部署和调整网络以满足需求。
:::
![3GPP TS 23.501](../asset/op5gc/1675913512544-04ef79d2-764e-4cc3-b9d6-ff4216de3edb.png#averageHue=%23f5f5f5&clientId=u8416fa2b-76b8-4&from=markdown&id=wGbRp&originHeight=378&originWidth=812&originalType=url&ratio=1&rotation=0&showTitle=true&status=done&style=none&taskId=u6b23362a-e542-4009-aa2a-281b28e7def&title=3GPP%20TS%2023.501 "3GPP TS 23.501")
# 项目分析
在OAI Release v1.4.0版本中实现了如下NFs：

- [Access and Mobility Management Function](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-amf/-/tree/develop) (AMF)
- [Session Management Function](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-smf/-/tree/develop) (SMF)
- [User Plane Function](https://github.com/OPENAIRINTERFACE/openair-spgwu-tiny) (UPF) 
- [Network Repository Function](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-nrf/-/tree/develop) (NRF)
- [Authentication Server Function](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-ausf) (AUSF)
- [Unified Data Management](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-udm) (UDM)
- [Unified Data Repository](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-udr) (UDR)
- [Network Slice Selection Function](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-nssf) (NSSF)

以上NF实现了大部分3GPP TS 23.501标准中确定的网络功能，并已足够构建一个可用的5G核心网络。
根据OAI官网描述，这些核心网络功能实现经过了以下多种测试工具和gNB以及RAN的测试和验证，具有良好的可用性。

- 专业测试工具：[Mobileum dsTest](https://www.developingsolutions.com/products/about-dstest/)，[ng4T](https://www.ng4t.com/)
- 商用 gNBs (including [Amarisoft](https://www.amarisoft.com/products/custom-projects/) gNB, [Baicell](https://na.baicells.com/) gNB with COST UE)
- OAI gNB with COTS UE (Quectel/SIMcom modules)
- 开源 RAN 模拟器 ([UERANSIM](https://github.com/aligungr/UERANSIM), [Gnbsim](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed/-/blob/master/docs/DEPLOY_SA5G_WITH_GNBSIM.md), [My-5GRANTester](https://github.com/my5G/my5G-RANTester))

为了最大化降低核心网络中数据处理的延迟，OAI中的所有NF组件都使用C和C++编写，OAI代码托管在自建的Gitlab仓库中。
![](../asset/op5gc/1675925379038-96311806-3ba6-4c45-a4f2-fbf7311d35b7.png#averageHue=%23fefefe&clientId=u8416fa2b-76b8-4&from=markdown&id=ITdUB&originHeight=873&originWidth=1221&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u48e583f7-3c30-4fa2-96e0-3dacf5ec689&title=)
# 活跃度
由于是有赞助商支持的非营利性组织，OAI在社区、宣发方面做的更为专业。
对于各种重要新闻OAI会通过新闻通知
![](../asset/op5gc/1675926950341-79b6e624-f0ba-4357-ae17-8caed69d6117.png#averageHue=%23fafaf9&clientId=u8416fa2b-76b8-4&from=markdown&height=746&id=MfBn0&originHeight=995&originWidth=1281&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=uf07b2c3c-28d0-45f8-983c-73106acb3a2&title=&width=961)
重要功能更新也会通过研讨会的形式发布
![](../asset/op5gc/1675927034491-a4c984ac-5e19-4162-9826-6c5701605538.png#averageHue=%23e5cba4&clientId=u8416fa2b-76b8-4&from=markdown&id=liGXo&originHeight=968&originWidth=1430&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u2e428193-90d6-4f20-b309-f05dfd0bb04&title=)

# 可用性及限制
专业团队的开源项目各方面配置也比较全面，每一个NF代码都配套了对应的文档说明、构建工具、持续集成代码以及容器化部署教程。无需社区支持在可用性方面已经做的非常优秀。
![](../asset/op5gc/1675928242825-ad008961-cf44-4f6a-b686-413f7a39a473.png#averageHue=%23fefdfc&clientId=u8416fa2b-76b8-4&from=markdown&id=FGCq7&originHeight=425&originWidth=1212&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=u1be8cda9-f83c-4472-8d73-e8862ccf189&title=)
官方提供了多种环境的应用部署教程，包括裸机/虚拟机、容器、OpenShift以及Kubernets等。
对于以云原生的方式部署核心网，官方也贴心的发布了教程及镜像以供使用，详见[OpenAirInterface on Kubernetes](https://github.com/OPENAIRINTERFACE/openair-k8s)。
官方甚至非常贴心的在视频网站开设频道介绍了旗下项目的各种实践。
![](../asset/op5gc/1675928758032-e2db4e6a-e1be-4aea-b16e-d4cc18318109.png#averageHue=%23f0eeed&clientId=u8416fa2b-76b8-4&from=markdown&id=qXU0q&originHeight=865&originWidth=1631&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=uf0015962-5aac-4520-b49f-35f864de55b&title=)
# 总结
OAI的核心网项目似乎在调研的几个项目中专业程度首屈一指，开发团队的规模也是最大的。各种部署实践的教程也都能在官网和开源社区找到。
但目前尚不清楚项目体量大小，不确定是否会出现太大或太重的情况。
