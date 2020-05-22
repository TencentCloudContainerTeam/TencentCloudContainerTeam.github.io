---
title: 揭秘 Kubernetes attach/detach controller 逻辑漏洞致使 pod 启动失败
date: 2020/05/13 21:00:00
---

作者： [蔡靖](https://github.com/ivan-cai)

### 前言

本文主要通过深入学习k8s attach/detach controller源码，了解现网案例发现的attach/detach controller bug发生的原委，并给出解决方案。

看完本文你也将学习到：
- attach/detach controller的主要数据结构有哪些，保存什么数据，数据从哪来，到哪去等等；
- k8s attach/detach volume的详细流程，如何判断volume是否需要attach/detach，attach/detach controller和kubelet(volume manager)如何协同工作等等。


### 现网案例现象
我们首先了解下现网案例的问题和现象；然后去深入理解ad controller维护的数据结构；之后根据数据结构与ad controller的代码逻辑，再来详细分析现网案例出现的原因和解决方案。从而深入理解整个ad controller。

#### 问题描述

- 一个statefulsets(sts)引用了多个pvc cbs，我们更新sts时，删除旧pod，创建新pod，此时如果删除旧pod时cbs detach失败，且创建的新pod调度到和旧pod相同的节点，就可能会让这些pod一直处于`ContainerCreating` 。


#### 现象

- `kubectl describe pod`

![enter image description here](https://main.qcloudimg.com/raw/e502a73a01436daa323e6747ae151b67.png)


-  kubelet log

![enter image description here](https://main.qcloudimg.com/raw/376e655665a8738810a78370f9bd3bee.png)


- `kubectl get node xxx -oyaml` 的`volumesAttached`和`volumesInUse`

```
volumesAttached:
  - devicePath: /dev/disk/by-id/virtio-disk-6w87j3wv
    name: kubernetes.io/qcloud-cbs/disk-6w87j3wv
volumesInUse:
  - kubernetes.io/qcloud-cbs/disk-6w87j3wv
  - kubernetes.io/qcloud-cbs/disk-7bfqsft5
```

### k8s存储简述

k8s中attach/detach controller负责存储插件的attach/detach。本文结合现网出现的一个案例来分析ad controller的源码逻辑，该案例是因k8s的ad controller bug导致的pod创建失败。

k8s中涉及存储的组件主要有：attach/detach controller、pv controller、volume manager、volume plugins、scheduler。每个组件分工明确：

- **attach/detach controller**：负责对volume进行attach/detach
- **pv controller**：负责处理pv/pvc对象，包括pv的provision/delete（cbs intree的provisioner设计成了external provisioner，独立的cbs-provisioner来负责cbs pv的provision/delete）
- **volume manager**：主要负责对volume进行mount/unmount
- **volume plugins**：包含k8s原生的和各厂商的的存储插件
  - 原生的包括：emptydir、hostpath、flexvolume、csi等
  - 各厂商的包括：aws-ebs、azure、我们的cbs等
- **scheduler**：涉及到volume的调度。比如对ebs、csi等的单node最大可attach磁盘数量的predicate策略

![enter image description here](https://main.qcloudimg.com/raw/9acf4b3722150ca3c315e19cd0b17132.png)


控制器模式是k8s非常重要的概念，一般一个controller会去管理一个或多个API对象，以让对象从实际状态/当前状态趋近于期望状态。

所以attach/detach controller的作用其实就是去attach期望被attach的volume，detach期望被detach的volume。

后续attach/detach controller简称ad controller。


### ad controller数据结构

对于ad controller来说，理解了其内部的数据结构，再去理解逻辑就事半功倍。ad controller在内存中维护2个数据结构：

1. `actualStateOfWorld`  ——   表征实际状态（后面简称asw）
2. `desiredStateOfWorld`  ——    表征期望状态（后面简称dsw）

很明显，对于声明式API来说，是需要随时比对实际状态和期望状态的，所以ad controller中就用了2个数据结构来分别表征实际状态和期望状态。



#### actualStateOfWorld

`actualStateOfWorld`  包含2个map：

- `attachedVolumes`： 包含了那些ad controller认为被成功attach到nodes上的volumes
- `nodesToUpdateStatusFor`： 包含要更新`node.Status.VolumesAttached` 的nodes



##### attachedVolumes

###### 如何填充数据？

1、在启动ad controller时，会populate asw，此时会list集群内所有node对象，然后用这些node对象的`node.Status.VolumesAttached` 去填充`attachedVolumes`。

2、之后只要有需要attach的volume被成功attach了，就会调用`MarkVolumeAsAttached`（`GenerateAttachVolumeFunc` 中）来填充到`attachedVolumes中`。

###### 如何删除数据？

1、只有在volume被detach成功后，才会把相关的volume从`attachedVolumes`中删掉。（`GenerateDetachVolumeFunc` 中调用`MarkVolumeDetached`)



##### nodesToUpdateStatusFor

######  如何填充数据？

1、detach volume失败后，将volume add back到`nodesToUpdateStatusFor`

​	- `GenerateDetachVolumeFunc` 中调用`AddVolumeToReportAsAttached`

###### 如何删除数据？

1、在detach volume之前会先调用`RemoveVolumeFromReportAsAttached` 从`nodesToUpdateStatusFor`中先删除该volume相关信息



#### desiredStateOfWorld

`desiredStateOfWorld` 中维护了一个map：

`nodesManaged`：包含被ad controller管理的nodes，以及期望attach到这些node上的volumes。



##### nodesManaged

###### 如何填充数据？

1、在启动ad controller时，会populate asw，list集群内所有node对象，然后把由ad controller管理的node填充到`nodesManaged`

2、ad controller的`nodeInformer` watch到node有更新也会把node填充到`nodesManaged`

3、另外在populate dsw和`podInformer` watch到pod有变化（add, update）时，往`nodesManaged` 中填充volume和pod的信息

4、`desiredStateOfWorldPopulator` 中也会周期性地去找出需要被add的pod，此时也会把相应的volume和pod填充到`nodesManaged` 

###### 如何删除数据？

1、当删除node时，ad controller中的`nodeInformer` watch到变化会从dsw的`nodesManaged` 中删除相应的node

2、当ad controller中的`podInformer` watch到pod的删除时，会从`nodesManaged` 中删除相应的volume和pod

3、`desiredStateOfWorldPopulator` 中也会周期性地去找出需要被删除的pod，此时也会从`nodesManaged` 中删除相应的volume和pod





### ad controller流程简述

ad controller的逻辑比较简单：

1、首先，list集群内所有的node和pod，来populate `actualStateOfWorld` (`attachedVolumes` )和`desiredStateOfWorld` (`nodesManaged`)

2、然后，单独开个goroutine运行`reconciler`，通过触发attach, detach操作周期性地去reconcile asw（实际状态）和dws（期望状态）

 - 触发attach，detach操作也就是，detach该被detach的volume，attach该被attach的volume

3、之后，又单独开个goroutine运行`DesiredStateOfWorldPopulator` ，定期去验证dsw中的pods是否依然存在，如果不存在就从dsw中删除





### 现网案例

接下来结合上面所说的现网案例，来详细看看`reconciler`的逻辑。


#### 案例初步分析

- 从pod的事件可以看出来：ad controller认为cbs attach成功了，然后kubelet没有mount成功。
- ***但是***从kubelet日志却发现`Volume not attached according to node status` ，也就是说kubelet认为cbs没有按照node的状态去挂载。这个从node info也可以得到证实：`volumesAttached` 中的确没有这个cbs盘（disk-7bfqsft5）。
- node info中还有个现象：`volumesInUse` 中还有这个cbs。说明没有unmount成功

很明显，cbs要能被pod成功使用，需要ad controller和volume manager的协同工作。所以这个问题的定位首先要明确：

1. volume manager为什么认为volume没有按照node状态挂载，ad controller却认为volume attch成功了？
2. `volumesAttached`和`volumesInUse` 在ad controller和kubelet之间充当什么角色？

这里只对分析volume manager做简要分析。

- 根据`Volume not attached according to node status` 在代码中找到对应的位置，发现在`GenerateVerifyControllerAttachedVolumeFunc` 中。仔细看代码逻辑，会发现
  - volume manager的reconciler会先确认该被unmount的volume被unmount掉
  - 然后确认该被mount的volume被mount
    - 此时会先从volume manager的dsw缓存中获取要被mount的volumes（`volumesToMount`的`podsToMount` ）
    - 然后遍历，验证每个`volumeToMount`是否已经attach了
      - `这个volumeToMount`是由`podManager`中的`podInformer`加入到相应内存中，然后`desiredStateOfWorldPopulator`周期性同步到dsw中的
    - 验证逻辑中，在`GenerateVerifyControllerAttachedVolumeFunc`中会去遍历本节点的`node.Status.VolumesAttached`，如果没有找到就报错（`Volume not attached according to node status`）
- 所以可以看出来，***volume manager就是根据volume是否存在于`node.Status.VolumesAttached` 中来判断volume有无被attach成功***。
- 那谁去填充`node.Status.VolumesAttached` ？ad controller的数据结构`nodesToUpdateStatusFor` 就是用来存储要更新到`node.Status.VolumesAttached` 上的数据的。
- 所以，***如果ad controller那边没有更新`node.Status.VolumesAttached`，而又新建了pod，`desiredStateOfWorldPopulator` 从podManager中的内存把新建pod引用的volume同步到了`volumesToMount`中，在验证volume是否attach时，就会报错（Volume not attached according to node status）***
  - 当然，之后由于kublet的syncLoop里面会调用`WaitForAttachAndMount` 去等待volumeattach和mount成功，由于前面一直无法成功，等待超时，才会有会面`timeout expired` 的报错

所以接下来主要需要看为什么ad controller那边没有更新`node.Status.VolumesAttached`。



#### ad controller的`reconciler`详解

接下来详细分析下ad controller的逻辑，看看为什么会没有更新`node.Status.VolumesAttached`，但从事件看ad controller却又认为volume已经挂载成功。

从[流程简述](#流程简述)中表述可见，ad controller主要逻辑是在`reconciler`中。

- `reconciler`定时去运行`reconciliationLoopFunc`，周期为100ms。

- `reconciliationLoopFunc`的主要逻辑在`reconcile()`中：

  1. 首先，确保该被detach的volume被detach掉

     - 遍历asw中的`attachedVolumes`，对于每个volume，判断其是否存在于dsw中
       - 根据nodeName去dsw.nodesManaged中判断node是否存在
       - 存在的话，再根据volumeName判断volume是否存在
     - 如果volume存在于asw，且不存在于dsw，则意味着需要进行detach
     - 之后，根据`node.Status.VolumesInUse`来判断volume是否已经unmount完成，unmount完成或者等待6min timeout时间到后，会继续detach逻辑
     - 在执行detach volume之前，会先调用`RemoveVolumeFromReportAsAttached`从asw的`nodesToUpdateStatusFor`中去删除要detach的volume
     - 然后patch node，也就等于从`node.status.VolumesAttached`删除这个volume
     - 之后进行detach，detach失败主要分2种
       - 如果真正执行了`volumePlugin`的具体实现`DetachVolume`失败，会把volume add back到`nodesToUpdateStatusFor`（之后在attach逻辑结束后，会再次patch node）
       - 如果是operator_excutor判断还没到backoff周期，就会返回`backoffError`，直接跳过`DetachVolume`
     - backoff周期起始为500ms，之后指数递增至2min2s。已经detach失败了的volume，在每个周期期间进入detach逻辑都会直接返回`backoffError`

  2. 之后，确保该被attach的volume被attach成功

     - 遍历dsw的`nodesManaged`，判断volume是否已经被attach到该node，如果已经被attach到该node，则跳过attach操作

     - 去asw.attachedVolumes中判断是否存在，若不存在就认为没有attach到node
       - 若存在，再判断node，node也匹配就返回`attachedConfirmed`


     - 而`attachedConfirmed`是由asw中`AddVolumeNode`去设置的，`MarkVolumeAsAttached`设置为true。（true即代表该volume已经被attach到该node了）
     	- 之后判断是否禁止多挂载，再由operator_excutor去执行attach

    3. 最后，`UpdateNodeStatuses`去更新node status



#### 案例详细分析

- 前提
  - volume detach失败
  - sts+cbs(pvc)，pod recreate前后调度到相同的node
- 涉及k8s组件
	- ad controller
	- kubelet(volume namager)
- ad controller和kubelet(volume namager)通过字段`node.status.VolumesAttached`交互。
	- ad controller为`node.status.VolumesAttached`新增或删除volume，新增表明已挂载，删除表明已删除
	- kubelet(volume manager)需要验证新建pod中的(pvc的)volume是否挂载成功，存在于`node.status.VolumesAttached`中，则表明验证volume已挂载成功；不存在，则表明还未挂载成功。
- 以下是整个过程：
1. 首先，删除pod时，由于某种原因cbs detach失败，失败后就会backoff重试。
   1. 由于detach失败，该volume也不会从asw的`attachedVolumes`中删除
2. 由于detach时，
   1. 先从`node.status.VolumesAttached`中删除volume，之后才去执行detach
   2. detach时返回`backoffError`不会把该volumeadd back `node.status.VolumesAttached`
3. 之后，我们在backoff周期中（假如就为第一个周期的500ms中间）再次创建sts，pod被调度到之前的node
4. 而pod一旦被创建，就会被添加到dsw的`nodesManaged`(nodeName和volumeName都没变)
5. reconcile()中的第2步，会去判断volume是否被attach，此时发现该volume同时存在于asw和dws中，并且由于detach失败，也会在检测时发现还是attach，从而设置`attachedConfirmed`为true
6. ad controller就认为该volume被attach成功了
7. reconcile()中第1步的detach逻辑进行判断时，发现要detach的volume已经存在于`dsw.nodesManaged`了（由于nodeName和volumeName都没变），这样volume同时存在于asw和dsw中了，实际状态和期望状态一致，被认为就不需要进行detach了。
8. 这样，该volume之后就再也不会被add back到`node.status.VolumesAttached`。所以就出现了现象中的node info中没有该volume，而ad controller又认为该volume被attach成功了
9. 由于kubelet(volume manager)与controller manager是异步的，而它们之间交互是依据`node.status.VolumesAttached` ，所以volume manager在验证volume是否attach成功，发现`node.status.VolumesAttached`中没有这个voume，也就认为没有attach成功，所以就有了现象中的报错`Volume not attached according to node status`
10. 之后kubelet的`syncPod`在等待pod所有的volume attach和mount成功时，就超时了(现象中的另一个报错`timeout expired wating...`)。
11. 所以pod一直处于`ContainerCreating`



#### 小结

- 所以，该案例出现的原因是：
  - sts+cbs，pod recreate时间被调度到相同的node
  - 由于detach失败，backoff期间创建sts/pod，致使ad controller中的dsw和asw数据一致（此时该volume由于没有被detach成功而确实处于attach状态），从而导致ad controller认为不再需要去detach该volume。
  - 又由于detach时，是先从`node.status.VolumesAttached`中删除该volume，再去执行真正的`DetachVolume`。backoff期间直接返回`backoffError`，跳过`DetachVolume`，不会add back
  - 之后，ad controller因volume已经处于attach状态，认为不再需要被attach，就不会再向`node.status.VolumesAttached`中添加该volume
  - 最后，kubelet与ad controller交互就通过`node.status.VolumesAttached`，所以kubelet认为没有attach成功，新创建的pod就一直处于`ContianerCreating`了
- 据此，我们可以发现关键点在于`node.status.VolumesAttached`和以下两个逻辑：
  1. detach时backoffError，不会add back
  2. detach是先删除，失败再add back
- 所以只要想办法能在任何情况下add back就不会有问题了。根据以上两个逻辑就对应有以下2种解决方案，**推荐使用方案2**：
  1. backoffError时，也add back
	- [pr #72914](https://github.com/kubernetes/kubernetes/pull/72914)
		- 但这种方式有个缺点：patch node的请求数增加了10+次/(s * volume)
  2. 一进入detach逻辑就判断是否backoffError（处于backoff周期中），是就跳过之后所有detach逻辑，不删除就不需要add back了。
	- [pr #88572](https://github.com/kubernetes/kubernetes/pull/88572)
		- 这个方案能避免方案1的问题，且会进一步减少请求apiserver的次数，且改动也不多



### 总结
- AD Controller负责存储的Attach、Detach。通过比较asw和dsw来判断是否需要attach/detach。最终attach和detach结果会体现在`node.status.VolumesAttached`。
- 以上现网案例出现的现象，是k8s ad controller的bug导致，目前社区并未修复。
	- 现象出现的原因主要是：
		- 先删除旧pod过程中detach失败，而在detach失败的backoff周期中创建新pod，此时由于ad controller逻辑bug，导致volume被从`node.status.VolumesAttached`中删除，从而导致创建新pod时，kubelet检查时认为该volume没有attach成功，致使pod就一直处于`ContianerCreating`。
	- 而现象的解决方案，推荐使用[pr #88572](https://github.com/kubernetes/kubernetes/pull/88572)。目前TKE已经有该方案的稳定运行版本，在灰度中。








