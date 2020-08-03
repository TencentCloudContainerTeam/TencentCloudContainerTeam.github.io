---
title: 大规模使用ConfigMap卷的负载分析及缓解方案
date: 2020/04/24 15:00:00
---

作者: [李波](https://github.com/borgerli)

## 简介

有客户反馈在大集群(几千节点)中大量使用ConfigMap卷时，会给集群带来很大负载和压力，这里我们分析下原因以及缓解方案。

## Kubelet如何管理ConfigMap

我们先来看下Kubelet是如何管理ConfigMap的。

Kubelet在启动的时候，会创建ConfigMapManager（以及SecretManager），用来管理本机运行的Pod用到的ConfigMap（及Secret，下面只讨论ConfigMap）对象，功能包括获取及更新这些对象的内容，以及为其他组件比如VolumeManager提供获取这些对象内容的服务。

那Kubelet是如何获取和更新ConfigMap呢？ k8s提供了三种检测资源更新的策略(`ResourceChangeDetectionStrategy`)

### WatchChangeDetectionStrategy(Watch)

这是`1.14+`的默认策略。

看名字，这个策略使用K8s经典的ListWatch模式。在Pod创建时，对每个引用到的ConfigMap，都会先从ApiServer缓存（指定ResourceVersion="0"）获取，然后对后续变化进行Watch。

```go
// pkg/kubelet/util/manager/watch_based_manager.go
func (c *objectCache) newReflector(namespace, name string) *objectCacheItem {
	fieldSelector := fields.Set{"metadata.name": name}.AsSelector().String()
	listFunc := func(options metav1.ListOptions) (runtime.Object, error) {
		options.FieldSelector = fieldSelector
		return c.listObject(namespace, options)
	}
	watchFunc := func(options metav1.ListOptions) (watch.Interface, error) {
		options.FieldSelector = fieldSelector
		return c.watchObject(namespace, options)
	}
	store := c.newStore()
	reflector := cache.NewNamedReflector(
		fmt.Sprintf("object-%q/%q", namespace, name),
		&cache.ListWatch{ListFunc: listFunc, WatchFunc: watchFunc},
		c.newObject(),
		store,
		0,
	)
	...
	...
```

重点强调下，是**对每一个ConfigMap都会创建一个Watch**。如果大量使用CongiMap，并且集群规模很大，假设平均每个节点有100个ConfigMap，集群有2000个节点，就会创建20w个watch。经过测试（测试结果如下图，20w个watch），单纯大量的watch会对ApiServer造成一定的内存压力，对Etcd则基本没有压力。

#### ListWatch压力测试

!['ListWatch压力测试结果'](https://github.com/TencentCloudContainerTeam/TencentCloudContainerTeam.github.io/raw/develop/source/_posts/res/images/configmap-5node-20w-watch.png)

测试采用单节点的ApiServer（16核32G）和单节点的Etcd，并停止所有（共5个）节点kubelet服务以及删除所有非kube-system的负载，并把5个节点作为客户端，每个有间隔的发起4w个ListWatch。
从上图的测试结果，可以看到在20w ListWatch创建期间，ApiServer的内存增长到20G左右，CPU使用率在25%左右（创建完成后，使用率降回原来水平），连接数增持长并稳定到965个左右，而Etcd的内存，CPU核连接数无明显变化。粗略计算，每个watch占用**100KB**左右的内存。

### TTLCacheChangeDetectionStrategy(Cache)

这是`1.10~1.13`版本的默认策略，`1.12+`可以通过配置文件进行定制。

看名字，这是带TTL的缓存方式。第一次获取时，从ApiServer获取最新内容，超过TTL后，如果读取ConfigMap，会从ApiServer缓存获取(Get请求指定ResouceVersion=0)进行刷新，以减小对ApiServer和Etcd的压力。
```go
// pkg/kubelet/util/manager/cache_based_manager.go
func (s *objectStore) Get(namespace, name string) (runtime.Object, error) {
...
	if data.err != nil || !fresh {
		klog.V(1).Infof("data is null or object is not fresh: err=%v, fresh=%v", fresh)
		opts := metav1.GetOptions{}
		if data.object != nil && data.err == nil {
			util.FromApiserverCache(&opts) //opts.ResourceVersion = "0"
		}

		object, err := s.getObject(namespace, name, opts)
...
```

TTL时间首先会从节点的`Annotation["node.alpha.kubernetes.io/ttl"]`获取，如果节点没有设置，那么会使用默认值1分钟。

`node.alpha.kubernetes.io/ttl`由kube-controller-manager中的TTLController根据集群节点数自动设置，具体规则如下(例如100个节点及以下规模的集群，ttl是0s；随着集群规模变大，节点数大于100小于500时，节点ttl变为15s；当集群规模超过100又减小，少于90个节点时，节点的ttl又变回0s)：

```go
// pkg/controller/ttl/ttl_controller.go
	ttlBoundaries = []ttlBoundary{
		{sizeMin: 0, sizeMax: 100, ttlSeconds: 0},
		{sizeMin: 90, sizeMax: 500, ttlSeconds: 15},
		{sizeMin: 450, sizeMax: 1000, ttlSeconds: 30},
		{sizeMin: 900, sizeMax: 2000, ttlSeconds: 60},
		{sizeMin: 1800, sizeMax: 10000, ttlSeconds: 300},
		{sizeMin: 9000, sizeMax: math.MaxInt32, ttlSeconds: 600},
	}
```

### GetChangeDetectionStrategy(Get)

这是最简单直接粗暴的方式，每次获取ConfigMap时，都访问ApiServer从Etcd读取最新版本。

```go
// pkg/kubelet/configmap/configmap_manager.go
func (s *simpleConfigMapManager) GetConfigMap(namespace, name string) (*v1.ConfigMap, error) {
	return s.kubeClient.CoreV1().ConfigMaps(namespace).Get(name, metav1.GetOptions{})
}

```

## ConfigMap卷的自动更新机制

Kubelet在Pod创建成功后，会把Pod放到podWorker的工作队列，并指定延迟1分钟(`--sync-frequency`，默认1m)才能出队列被获取。
Kubelet在sync逻辑中，会在延迟过后取到Pod进行同步，包括同步Volume状态。VolumeManager在同步Volume时会看它的类型是否需要重新挂载(`RequiresRemount() bool`)，`ConfigMap`、`Secret`、`downwardAPI`及`Projected`四种VolumePlugin，这个方法都返回`true`，需要重新挂载。

因此每隔1分钟多，Kubelet都会访问ConfigMapManager，去获取本机Pod使用的ConfigMap的最新内容。这个操作对于Watch类型的策略，没有影响，不会对ApiServer及Etcd带来额外的压力；对于ttl很小的Cache及Get类型的策略，会给ApiServer及Etcd带来压力。


## 大集群方案
从上面的分析看，一般小规模的集群或者ConfigMap（及Secret）用量不大的集群，可以使用默认的Watch策略。如果集群规模比较大，并且大量使用ConfigMap，默认的Watch策略会对ApiServer带来内存压力。在实际生产集群，ApiServer除了处理这些watch，还会执行很多其他任务，相互之间共享抢占系统资源，会加重和放大对ApiServer的负载，影响服务。

同时，实际上我们很多应用并不需要通过修改ConfigMap动态更新配置的功能，一方面在大集群时会带来不必要的压力，另一方面，如1.18的这个[KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-storage/20191117-immutable-secrets-configmaps.md)所考虑的，实时更新ConfigMap或者Secret，如果内容出现错误，会导致应用异常，在配置发生变化时，更推荐采用滚动更新的方式来更新应用。

在大集群时，我们可以怎么使用和管理ConfigMap，来减轻对集群的负载压力呢？

### 1.18版本

社区也注意到了这个问题（刚才提到的[KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-storage/20191117-immutable-secrets-configmaps.md)），增加了一个新的特性`ImmutableEphemeralVolumes`，允许用户设置ConfigMap（及Secrets）为不可变（`immutable: true`），这样Kubelet就不会去Watch这些ConfigMap的变化了。

```go
...
		if utilfeature.DefaultFeatureGate.Enabled(features.ImmutableEphemeralVolumes) && c.isImmutable(object) {
			if item.stop() {
				klog.V(4).Infof("Stopped watching for changes of %q/%q - object is immutable", namespace, name)
			}
		}
...		
```

#### 开启ImmutableEphemeralVolumes

ImmutableEphemeralVolumes是alpha特性，需要设置kubelet参数开启它：

```bash
--feature-gates=ImmutableEphemeralVolumes=true
```

#### ConfigMap设置为不可变

ConfigMap设置`immutable`为true

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-cm
data:
  name: tencent
immutable: true
```



### 之前版本

在1.18之前的版本，我们可以使用`Cache`策略来代替`Watch`。

1. 关闭`TTLController`: kube-controller-manager启动参数增加 `--controllers=-ttl,*`，重启。

2.  配置所有节点Kubelet使用`Cache`策略: 

 - 创建`/etc/kubernetes/kubelet.conf`，内容如下：

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
configMapAndSecretChangeDetectionStrategy: Cache
```
 - kubelet增加参数: `--config=/etc/kubernetes/kubelet.conf`，重启
3.  设置所有节点的ttl为期望值，比如1000天:  `kubectl annotate node <node> node.alpha.kubernetes.io/ttl=86400000 --overwrite`
    。设置1000天并不是1000天内真的不更新。在Kubelet新建Pod时，它所引用的ConfigMap的cache都会被重置和更新。