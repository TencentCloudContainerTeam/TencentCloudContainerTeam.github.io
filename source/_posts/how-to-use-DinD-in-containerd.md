---
title: 在containerd集群中使用docker做镜像构建服务
date: 2020/12/08 12:00:00
---

作者：[李志宇](https://github.com/payall4u)

# 在containerd集群中使用docker做镜像构建服务

	在K8S集群中，某些CI/CD流水线业务可能需要使用docker来提供镜像打包服务，一些同学会想到利用宿主机上的docker，具体做法是把宿主机上docker的unix socket（/var/run/docker.sock)作为host path挂载到CI/CD的业务pod中，之后在容器里通过unix socket来调用宿主机上的docker进行构建。这种方法固然简单，照比真正意义上的Docker in Docker甚至还能节省资源，但这种做法可能会遇到一些问题：

- 无法运行在runtime是containerd的集群中
- 如果不加以控制可能会覆盖掉节点上已有的镜像
- 在需要修改docker daemon配置文件的情况下，可能会影响到其他业务
- 这种方式并不安全，特别是在多租户的场景下，当有特权的pod拿到了docker的unix socket之后，pod中的容器不只可以调用宿主机的docker构建镜像，甚至可以删除已有镜像或容器，更有甚者可以通过docker exec接口操作其他容器

	第一个问题最近很多同学都会提到，因为Kubernetes在官方博客中宣布要在1.22版本之后弃用docker，这之后可能部分同学就会把业务转投到containerd上。对于某些想用containerd集群，而不想改变CI/CD业务流程仍使用docker做构建构成一部分的话，可以通过在原有pod上添加dind容器作为sidecar或者使用DaemonSet在节点上部署专门用来构建镜像的docker服务。

## 使用DinD作为Pod的sidecar

    DinD（Docker in Docker）具体的原理可以参照[DinD](https://hub.docker.com/_/docker)，下面的例子中是给clean-ci容器添加一个sidecar，并通过emptyDir让clean-ci容器可以通过unix socket访问到DinD容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: clean-ci
spec:
  containers:
  - name: dind
    image: 'docker:stable-dind'
    command:
    - dockerd
    - --host=unix:///var/run/docker.sock
    - --host=tcp://0.0.0.0:8000
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /var/run
      name: cache-dir
  - name: clean-ci
    image: 'docker:stable'
    command: ["/bin/sh"]
    args: ["-c", "docker info >/dev/null 2>&1; while [ $? -ne 0 ] ; do sleep 3; docker info >/dev/null 2>&1; done; docker pull library/busybox:latest; docker save -o busybox-latest.tar library/busybox:latest; docker rmi library/busybox:latest; while true; do sleep 86400; done"]
    volumeMounts:
    - mountPath: /var/run
      name: cache-dir
  volumes:
  - name: cache-dir
    emptyDir: {}
```

## 使用DaemonSet在每个containerd节点上部署Docker

	这种方式比较简单，直接在containerd集群中下发DaemonSet即可，当然不想污染节点上/var/run路径的同学们可以指定其他路径。

``` yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: docker-ci
spec:
  selector:
    matchLabels:
      app: docker-ci
  template:
    metadata:
      labels:
        app: docker-ci
    spec:
      containers:
      - name: docker-ci
        image: 'docker:stable-dind'
        command:
        - dockerd
        - --host=unix:///var/run/docker.sock
        - --host=tcp://0.0.0.0:8000
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /var/run
          name: host
      volumes:
      - name: host
        hostPath:
          path: /var/run
---
apiVersion: v1
kind: Pod
metadata:
  name: clean-ci
spec:
  containers:
  - name: clean-ci
    image: 'docker:stable'
    command: ["/bin/sh"]
    args: ["-c", "docker info >/dev/null 2>&1; while [ $? -ne 0 ] ; do sleep 3; docker info >/dev/null 2>&1; done; docker pull library/busybox:latest; docker save -o busybox-latest.tar library/busybox:latest; docker rmi library/busybox:latest; while true; do sleep 86400; done"]
    volumeMounts:
    - mountPath: /var/run
      name: host
  volumes:
  - name: host
    hostPath:
      path: /var/run
```
