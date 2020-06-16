---
title: "TKE 集群组建最佳实践"
date: 2020/06/16 09:00:00
---

作者: [陈鹏](https://imroc.io/)

## Kubernetes 版本

K8S 版本迭代比较快，新版本通常包含许多 bug 修复和新功能，旧版本逐渐淘汰，建议创建集群时选择当前 TKE 支持的最新版本，后续出新版本后也是可以支持 Master 和 节点的版本升级的。

## 网络模式: GlobalRouter vs VPC-CNI

**GlobalRouter 模式架构:**

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/global-router.jpg)

* 基于 CNI 和 网桥实现的容器网络能力，容器路由直接通过 VPC 底层实现
* 容器与节点在同一网络平面，但网段不与 VPC 网段重叠，容器网段地址充裕 

**VPC-CNI 模式架构:**

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/vpc-cni.jpg)

* 基于 CNI 和 VPC 弹性网卡实现的容器网络能力，容器路由通过弹性网卡，性能相比 Global Router 约提高 10%
* 容器与节点在同一网络平面，网段在 VPC 网段内
* 支持 Pod 固定 IP

**对比:**

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/global-router-vs-vpc-cni.jpg)

**支持三种使用方式:**

* 创建集群时指定 GlobalRouter 模式
* 创建集群时指定 VPC-CNI 模式，后续所有 Pod 都必须使用 VPC-CNI 模式创建
* 创建集群时指定 GlobalRouter 模式，在需要使用 VPC-CNI 模式时为集群启用 VPC-CNI 的支持，即两种模式混用

**选型建议:**

* 绝大多数情况下应该选择 GlobalRouter，容器网段地址充裕，扩展性强，能适应规模较大的业务
* 如果后期部分业务需要用到 VPC-CNI 模式，可以在 GlobalRouter 集群再开启 VPC-CNI 支持，也就是 GlobalRouter 与 VPC-CNI 混用，仅对部分业务使用 VPC-CNI 模式
* 如果完全了解并接受 VPC-CNI 的各种限制，并且需要集群内所有 Pod 都用 VPC-CNI 模式，可以创建集群时选择 VPC-CNI 网络插件

> 参考官方文档 《如何选择容器服务网络模式》: https://cloud.tencent.com/document/product/457/41636

## 运行时: Docker vs Containerd

**Docker 作为运行时的架构:**

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/docker-as-runtime.jpg)

* kubelet 内置的 dockershim 模块帮傲娇的 docker 适配了 CRI 接口，然后 kubelet 自己调自己的 dockershim (通过 socket 文件)，然后 dockershim 再调 dockerd 接口 (Docker HTTP API)，接着 dockerd 还要再调 docker-containerd (gRPC) 来实现容器的创建与销毁等。
* 为什么调用链这么长？ K8S 一开始支持的就只是 Docker，后来引入了 CRI，将运行时抽象以支持多种运行时，而 Docker 跟 K8S 在一些方面有一定的竞争，不甘做小弟，也就没在 dockerd 层面实现 CRI 接口，所以 kubelet 为了让 dockerd 支持 CRI，就自己为 dockerd 实现了 CRI。docker 本身内部组件也模块化了，再加上一层 CRI 适配，调用链肯定就长了。

**Containerd 作为运行时的架构:**

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/containerd-as-runtime.jpg)

* containerd 1.1 之后，支持 CRI Plugin，即 containerd 自身这里就可以适配 CRI 接口。
* 相比 Docker 方案，调用链少了 dockershim 和 dockerd。

**对比:**

* containerd 方案由于绕过了 dockerd，调用链更短，组件更少，占用节点资源更少，绕过了 dockerd 本身的一些 bug，但 containerd 自身也还存在一些 bug (已修复一些，灰度中)。
* docker 方案历史比较悠久，相对更成熟，支持 docker api，功能丰富，符合大多数人的使用习惯。

**选型建议:**

* Docker 方案 相比 containerd 更成熟，如果对稳定性要求很高，建议 docker 方案
* 以下场景只能使用 docker: 
  * Docker in docker (通常在 CI 场景)
  * 节点上使用 docker 命令
  * 调用 docker API
* 没有以上场景建议使用 containerd

> 参考官方文档 《如何选择 Containerd 和 Docker》: https://cloud.tencent.com/document/product/457/35747

## Service 转发模式: iptables vs ipvs

先看看 Service 的转发原理:

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/service.jpg)

* 节点上的 kube-proxy 组件 watch apiserver，获取 Service 与 Endpoint，转化成 iptables 或 ipvs 规则并写到节点上
* 集群内的 client 去访问 Service (Cluster IP)，会被 iptable/ipvs 规则负载均衡到 Service 对应的后端 pod

**对比:**

* ipvs 模式性能更高，但也存在一些已知未解决的 bug
* iptables 模式更成熟稳定

**选型建议:**

* 对稳定性要求极高且 service 数量小于 2000，选 iptables
* 其余场景首选 ipvs

## 集群类型: 托管集群 vs 独立集群

**托管集群:**

* Master 组件用户不可见，由腾讯云托管
* 很多新功能也是会率先支持托管的集群
* Master 的计算资源会根据集群规模自动扩容
* 用户不需要为 Master 付费

**独立集群:**

* Master 组件用户可以完全掌控
* 用户需要为 Master 付费购买机器

**选型建议:**

* 一般推荐托管集群
* 如果希望能能够对 Master 完全掌控，可以使用独立集群 (比如对 Master 进行个性化定制实现高级功能)

## 节点操作系统

TKE 主要支持 Ubuntu 和 CentOS 两类发行版，带 “TKE-Optimized” 后缀用的是 TKE 定制优化版的内核，其它的是 linux 社区官方开源内核:

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/ubuntu.jpg)

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/centos.jpg)

**TKE-Optimized 的优势:**

* 基于内核社区长期支持的 4.14.105 版本定制
* 针对容器和云场景进行优化
* 计算、存储和网络子系统均经过性能优化
* 对内核缺陷修复支持较好
* 完全开源： https://github.com/Tencent/TencentOS-kernel

**选型建议:**

* 推荐 “TKE-Optimized”，稳定性和技术支持都比较好
* 如果需要更高版本内核，选非 “TKE-Optimized” 版本的操作系统

## 节点池

此特性当前正在灰度中，可申请开白名单使用。主要可用于批量管理节点：

* 节点 Label 与 Taint
* 节点组件启动参数
* 节点自定义启动脚本
* 操作系统与运行时 (暂未支持)

> 产品文档：https://cloud.tencent.com/document/product/457/43719

**适用场景:**

* 异构节点分组管理，减少管理成本
* 让集群更好支持复杂的调度规则 (Label, Taint)
* 频繁扩缩容节点，减少操作成本
* 节点日常维护(版本升级)

**用法举例:**

部分IO密集型业务需要高IO机型，为其创建一个节点池，配置机型并统一设置节点 Label 与 Taint，然后将 IO 密集型业务配置亲和性，选中 Label，使其调度到高 IO 机型的节点 (Taint 可以避免其它业务 Pod 调度上来)。

随着时间的推移，业务量快速上升，该 IO 密集型业务也需要更多的计算资源，在业务高峰时段，HPA 功能自动为该业务扩容了 Pod，而节点计算资源不够用，这时节点池的自动伸缩功能自动扩容了节点，扛住了流量高峰。

## 启动脚本

添加节点时通过自定义数据配置节点启动脚本 (可用于修改组件启动参数、内核参数等):

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/custom-script.jpg)

## 组件自定义参数

此特性当前也正在灰度中，可申请开白名单使用。

创建集群时可自定义 Master 组件部分启动参数:

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/custom-master-parameter.jpg)

添加节点时可自定义 kubelet 部分启动参数:

![](https://imroc.io/assets/blog/tke-best-practices-and-troubleshooting/custom-kubelet-parameter.jpg)
