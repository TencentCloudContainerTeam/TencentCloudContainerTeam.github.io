---
title: Google Borg 浅析
date: 2019/04/17 15:00:00
---

作者: [徐蓓](https://github.com/xiaoxubeii)

# Google Borg 浅析
笔者的工作主要涉及集群资源调度和混合部署，对相关技术和论文有所研究，包括 Google Borg、Kubernetes、Firmament 和 Kubernetes Poseidon 等。尤其是这篇《Large-scale cluster management at Google with Borg》令笔者受益匪浅。下面本人就结合生产场景，尝试对 Google Borg 做些分析和延展。

## Google Borg 简介

Google Borg 是一套资源管理系统，可用于管理和调度资源。在 Borg 中，资源的单位是 **Job** 和 **Task**。**Job** 包含一组 **Task**。**Task** 是 Borg 管理和调度的最小单元，它对应一组 Linux 进程。熟悉 Kubernetes 的读者，可以将 **Job** 和 **Task** 大致对应为 Kubernetes 的 **Service** 和 **Pod**。

在架构上，Borg 和 Kubernetes 类似，由 BorgMaster、Scheduler 和 Borglet 组成。

![](/images/15414871166556.jpg)

## Allocs
Borg Alloc 代表一组可用于运行 Task 的资源，如 CPU、内存、IO 和磁盘空间。它实际上是集群对物理资源的抽象。Alloc set 类似 Job，是一堆 Alloc 的集合。当一个 Alloc set 被创建时，一个或多个 Job 就可以运行在上面了。

## Priority 和 Quota
每个 Job 都可以设置 Priority。Priority 可用于标识 Job 的重要程度，并影响一些资源分配、调度和 Preemption 策略。比如在生产中，我们会将作业分为 Routine Job 和 Batch Job。Routine Job 为生产级的例行作业，优先级最高，它占用对应实际物理资源的 Alloc set。Batch Job 代表一些临时作业，优先级最低。当资源紧张时，集群会优先 Preempt Batch Job，将资源提供给 Routine Job 使用。这时 Preempted Batch Job 会回到调度队列等待重新调度。

Quota 代表资源配额，它约束 Job 的可用资源，比如 CPU、内存或磁盘。Quota 一般在调度之前进行检查。Job 若不满足，会立即在提交时被拒绝。生产中，我们一般依据实际物理资源配置 Routine Job Quota。这种方式可以确保 Routine Job 在 Quota 内一定有可用的资源。为了充分提升集群资源使用率，我们会将 Batch Job Quota 设置为无限，让它尽量去占用 Routine Job 的闲置资源，从而实现超卖。这方面内容后面会在再次详述。

## Schedule
调度是资源管理系统的核心功能，它直接决定了系统的“好坏”。在 Borg 中，Job 被提交后，Borgmaster 会将其放入一个 Pending Queue。Scheduler 异步地扫描队列，将 Task 调度到有充足资源的机器上。通常情况下，调度过程分为两个步骤：Filter 和 Score。Filter，或是 Feasibility Checking，用于判断机器是否满足 Task 的约束和限制，比如 Schedule Preference、Affinity 或 Resource Limit。Filter 结束后，就需要 Score 符合要求的机器，或称为 Weight。上述两个步骤完成后，Scheduler 就会挑选相应数量的机器调度给 Task 运行。实际上，选择合适的调度策略尤为重要。

这里可以拿一个生产集群举例。在初期，我们的调度系统采用的 Score 策略类似 Borg E-PVM，它的作用是将 Task 尽量均匀的调度到整个集群上。从正面效果上讲，这种策略分散了 Task 负载，并在一定程度上缩小了故障域。但从反面看，它也引发了资源碎片化的问题。由于我们底层环境是异构的，机器配置并不统一，并且 Task 配置和物理配置并无对应关系。这就造成一些配置过大的 Task 无法运行，由此在一定程度上降低了资源的分配率和使用率。为了应付此类问题，我们自研了新的 Score 策略，称之为 “Best Fillup”。它的原理是在调度 Task 时选择可用资源最少的机器，也就是尽量填满。不过这种策略的缺点显而易见：单台机器的负载会升高，从而增加 Bursty Load 的风险；不利于 Batch Job 运行；故障域会增加。

这篇论文，作者采用了一种被称为 hybrid 的方式，据说比第一种策略增加 3-5% 的效率。具体实现方式还有待后续研究。

## Utilization
资源管理系统的首要目标是提高资源使用率，Borg 亦是如此。不过由于过多的前置条件，诸如 Job 放置约束、负载尖峰、多样的机器配置和 Batch Job，导致不能仅选择 “average utilization” 作为策略指标。在 Borg 中，使用 **Cell Compaction** 作为评判基准。简述之就是：能承载给定负载的最小 Cell。

Borg 提供了一些提高 utilization 的思路和实践方法，有些是我们在生产中已经采用的，有些则非常值得我们学习和借鉴。
### Cell Sharing
Borg 发现，将各种优先级的 Task，比如 prod 和 non-prod 运行在共享的 Cell 中可以大幅度的提升资源利用率。

![](/images/15414743848812.jpg)

上面（a）图表明，采用 Task 隔离的部署方式会增加对机器的需求。图（b）是对额外机器需求的分布函数。图（a）和图（b）都清楚的表明了将 prod job 和 non-prod job 分开部署会消耗更多的物理资源。Borg 的经验是大约会新增 20-30% 左右。

个中原理也很好理解：prod job 通常会为应对负载尖峰申请较大资源，实际上这部分资源在多数时间里是闲置的。Borg 会定时回收这部分资源，并将之分配给 non-prod job 使用。在 Kubernetes 中，对应的概念是 request limit 和 limit。我们在生产中，一般设置 Prod job 的 Request limit 等于 limit，这样它就具有了最高的 Guaranteed Qos。该 QoS 使得 pod 在机器负载高时不至于被驱逐和 OOM。non-prod job 则不设置 request limit 和 limit，这使得它具有 BestEffort 级别的 QoS。kubelet 会在资源负载高时优先驱逐此类 Pod。这样也达到了和 Borg 类似的效果。

### Large cells
Borg 通过实验数据表明，小容量的 cell 通常比大容量的更占用物理资源。
![](/images/15414759002584.jpg)

这点对我们有和很重要的指导意义。通常情况下，我们会在设计集群时对容量问题感到犹豫不决。显而易见，小集群可以带来更高的隔离性、更小的故障域以及潜在风险。但随之带来的则是管理和架构复杂度的增加，以及更多的故障点。大集群的优缺点正好相反。在资源利用率这个指标上，我们凭直觉认为是大集群更优，但苦于无坚实的理论依据。Borg 的研究表明，大集群有利于增加资源利用率，这点对我们的决策很有帮助。

### Fine-grained resource requests
Borg 对资源细粒度分配的方法，目前已是主流，在此我就不再赘述。

### Resource reclamation
笔者感觉这部分内容帮助最大。熟悉 Kubernetes 的读者，应该对类似的概念很熟悉，也就是所谓的 request limit。job 在提交时需要指定 resource limit，它能确保内部的 task 有足够资源可以运行。有些用户会为 task 申请过大的资源，以应对可能的请求或计算的突增。但实际上，部分资源在多数时间内是闲置的。与其资源浪费，不如利用起来。这需要系统有较精确的预测机制，可以评估 task 对实际资源的需求，并将闲置资源回收以分配给低 priority 的任务，比如 batch job。上述过程在 Borg 中被称为 **resource reclamation**，对使用资源的评估则被称为 **reservation**。Borgmaster 会定期从 Borglet 收集 resource consumption，并执行 **reservation**。在初始阶段，reservation 等于 resource limit。随着 task 的运行，reservation 就变为了资源的实际使用量，外加 safety margin。

在 Borg 调度时，Scheduler 使用 resource limit 为 prod task 过滤和选择主机，这个过程并不依赖 reclaimed resource。从这个角度看，并不支持对 prod task 的资源超卖。但 non-prod task 则不同，它是占用已有 task 的 resource reservation。所以 non-prod task 会被调度到拥有 reclaimed resource 的机器上。

这种做法当然也是有一定风险的。若资源评估出现偏差，机器上的可用资源可能会被耗尽。在这种情况下，Borg 会杀死或者降级 non-prod task，prod task 则不会受到半分任何影响。

![](/images/15414862899318.jpg)

上图证实了这种策略的有效性。参照 Week 1 和 4 的 baseline，Week 2 和 3 在调整了 estimation algorithm 后，实际资源的 usage 与 reservation 的 gap 在显著缩小。在 Borg 的一个 median cell 中，有 20% 的负载是运行在 reclaimed resource 上。

相较于 Borg，Kubernetes 虽然有 resource limit 和 capacity 的概念，但却缺少动态 reclaim 机制。这会使得系统对低 priority task 的资源缺少行之有效的评估机制，从而引发系统负载问题。个人感觉这个功能对资源调度和提升资源使用率影响巨大，这部分内容也是笔者的工作重心

## Isolation
这部分内容虽十分重要，但对于我们的生产集群优先级不是很高，在此先略过。有兴趣的读者可以自行研究。

## 参考资料
* [Large-scale cluster management at Google with Borg]()
* [The evolution of cluster scheduler architectures](http://www.firmament.io/blog/scheduler-architectures.html)
* [poseidon](https://github.com/kubernetes-sigs/poseidon)
* [Poseidon design](https://docs.google.com/document/d/1VNoaw1GoRK-yop_Oqzn7wZhxMxvN3pdNjuaICjXLarA/edit?usp=sharing)
* [firemament](https://github.com/camsas/firmament)
