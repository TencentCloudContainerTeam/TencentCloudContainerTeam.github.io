---
title: K8s Network Policy Controller之Kube-router功能介绍
date: 2018/10/30 17:22:24
---

Author: [Jimmy Zhang](https://github.com/jimmy-zh) (张浩)

# Network Policy

[Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)是k8s提供的一种资源，用于定义基于pod的网络隔离策略。它描述了一组pod是否可以与其它组pod，以及其它network endpoints进行通信。

# Kube-router

- 官网:  [https://www.kube-router.io](https://www.kube-router.io)
- 项目:  [https://github.com/cloudnativelabs/kube-router](https://github.com/cloudnativelabs/kube-router)
- 目前最新版本：[v0.2.1](https://github.com/cloudnativelabs/kube-router/releases/tag/v0.2.1)

kube-router项目的三大功能：

- Pod Networking
- IPVS/LVS based service proxy  
- Network Policy Controller 

在腾讯云TKE上，Pod Networking功能由基于IAAS层VPC的高性能容器网络实现，service proxy功能由kube-proxy所支持的ipvs/iptables两种模式实现。建议在TKE上，只使用kube-router的Network Policy功能。

# 在TKE上部署kube-router

### 腾讯云提供的kube-router版本

腾讯云PAAS团队提供的镜像"ccr.ccs.tencentyun.com/library/kube-router:v1"基于官方的最新版本：[v0.2.1](https://github.com/cloudnativelabs/kube-router/releases/tag/v0.2.1)

在该项目的开发过程中，腾讯云PAAS团队积极参与社区，持续贡献了一些feature support和bug fix, 列表如下（均已被社区合并）：

- [https://github.com/cloudnativelabs/kube-router/pull/488](https://github.com/cloudnativelabs/kube-router/pull/488)
- [https://github.com/cloudnativelabs/kube-router/pull/498](https://github.com/cloudnativelabs/kube-router/pull/498)
- [https://github.com/cloudnativelabs/kube-router/pull/527](https://github.com/cloudnativelabs/kube-router/pull/527)
- [https://github.com/cloudnativelabs/kube-router/pull/529](https://github.com/cloudnativelabs/kube-router/pull/529)
- [https://github.com/cloudnativelabs/kube-router/pull/543](https://github.com/cloudnativelabs/kube-router/pull/543)

我们会继续贡献社区，并提供腾讯云镜像的版本升级。

### 部署kube-router

Daemonset yaml文件:

> [#kube-router-firewall-daemonset.yaml.zip#](https://ask.qcloudimg.com/draft/982360/9wn7eu0bek.zip)

在**能访问公网**，也能访问TKE集群apiserver的机器上，执行以下命令即可完成kube-router部署。

如果集群节点开通了公网IP，则可以直接在集群节点上执行以下命令。

如果集群节点没有开通公网IP, 则可以手动下载和粘贴yaml文件内容到节点, 保存为kube-router-firewall-daemonset.yaml，再执行最后的kubectl create命令。

```
wget https://ask.qcloudimg.com/draft/982360/9wn7eu0bek.zip
unzip 9wn7eu0bek.zip
kuebectl create -f kube-router-firewall-daemonset.yaml
```

### yaml文件内容和参数说明

kube-router-firewall-daemonset.yaml文件内容：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-router-cfg
  namespace: kube-system
  labels:
    tier: node
    k8s-app: kube-router
data:
  cni-conf.json: |
    {
      "name":"kubernetes",
      "type":"bridge",
      "bridge":"kube-bridge",
      "isDefaultGateway":true,
      "ipam": {
        "type":"host-local"
      }
    }
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-router
  namespace: kube-system
  labels:
    k8s-app: kube-router
spec:
  template:
    metadata:
      labels:
        k8s-app: kube-router
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      containers:
      - name: kube-router
        image: ccr.ccs.tencentyun.com/library/kube-router:v1
        args: ["--run-router=false", "--run-firewall=true", "--run-service-proxy=false", "--kubeconfig=/var/lib/kube-router/kubeconfig", "--iptables-sync-period=5m", "--cache-sync-timeout=3m"]
        securityContext:
          privileged: true
        imagePullPolicy: Always
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        livenessProbe:
          httpGet:
            path: /healthz
            port: 20244
          initialDelaySeconds: 10
          periodSeconds: 3
        volumeMounts:
        - name: lib-modules
          mountPath: /lib/modules
          readOnly: true
        - name: cni-conf-dir
          mountPath: /etc/cni/net.d
        - name: kubeconfig
          mountPath: /var/lib/kube-router/kubeconfig
          readOnly: true
      initContainers:
      - name: install-cni
        image: busybox
        imagePullPolicy: Always
        command:
        - /bin/sh
        - -c
        - set -e -x;
          if [ ! -f /etc/cni/net.d/10-kuberouter.conf ]; then
            TMP=/etc/cni/net.d/.tmp-kuberouter-cfg;
            cp /etc/kube-router/cni-conf.json ${TMP};
            mv ${TMP} /etc/cni/net.d/10-kuberouter.conf;
          fi
        volumeMounts:
        - name: cni-conf-dir
          mountPath: /etc/cni/net.d
        - name: kube-router-cfg
          mountPath: /etc/kube-router
      hostNetwork: true
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        operator: Exists
      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: cni-conf-dir
        hostPath:
          path: /etc/cni/net.d
      - name: kube-router-cfg
        configMap:
          name: kube-router-cfg
      - name: kubeconfig
        hostPath:
          path: /root/.kube/config
```

args说明：

1. "--run-router=false", "--run-firewall=true", "--run-service-proxy=false"：只加载firewall模块；
2. kubeconfig：用于指定master信息，映射到主机上的kubectl配置目录/root/.kube/config；
3. --iptables-sync-period=5m：指定定期同步iptables规则的间隔时间，根据准确性的要求设置，默认5m；
4. --cache-sync-timeout=3m：指定启动时将k8s资源做缓存的超时时间，默认1m；

# NetworkPolicy配置示例

### 1.nsa namespace下的pod可互相访问，而不能被其它任何pod访问

```
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
  name: npa
  namespace: nsa
spec:
  ingress: 
  - from:
    - podSelector: {} 
  podSelector: {} 
  policyTypes:
  - Ingress
```

### 2.nsa namespace下的pod不能被任何pod访问

```
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
  name: npa
  namespace: nsa
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

### 3.nsa namespace下的pod只在6379/TCP端口可以被带有标签app: nsb的namespace下的pod访问，而不能被其它任何pod访问

```
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
  name: npa
  namespace: nsa
spec:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          app: nsb
    ports:
    - protocol: TCP
      port: 6379
  podSelector: {}
  policyTypes:
  - Ingress
```

### 4.nsa namespace下的pod可以访问CIDR为14.215.0.0/16的network endpoint的5978/TCP端口，而不能访问其它任何network endpoints（此方式可以用来为集群内的服务开访问外部network endpoints的白名单）

```
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
  name: npa
  namespace: nsa
spec:
  egress:
  - to:
    - ipBlock:
        cidr: 14.215.0.0/16
    ports:
    - protocol: TCP
      port: 5978
  podSelector: {}
  policyTypes:
  - Egress
```

### 5.default namespace下的pod只在80/TCP端口可以被CIDR为14.215.0.0/16的network endpoint访问，而不能被其它任何network endpoints访问

```
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
  name: npd
  namespace: default
spec:
  ingress:
  - from:
    - ipBlock:
        cidr: 14.215.0.0/16
    ports:
    - protocol: TCP
      port: 80
  podSelector: {}
  policyTypes:
  - Ingress
```

# 附: 测试情况

| 用例名称 | 测试结果 |
|:----|:----|
| 不同namespace的pod互相隔离，同一namespace的pod互通 | 通过 |
| 不同namespace的pod互相隔离，同一namespace的pod隔离 | 通过 |
| 不同namespace的pod互相隔离，白名单指定B可以访问A | 通过 |
| 允许某个namespace访问集群外某个CIDR，其他外部IP全部隔离 | 通过 |
| 不同namespace的pod互相隔离，白名单指定B可以访问A中对应的pod以及端口 | 通过 |
| 以上用例，当source pod 和 destination pod在一个node上时，隔离是否生效 | 通过 |


功能测试用例

> [#kube-router测试用例.xlsx.zip#](https://ask.qcloudimg.com/draft/982360/dgs7x4hcly.zip)
