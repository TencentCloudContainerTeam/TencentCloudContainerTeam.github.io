---
title: Cgroup泄漏--潜藏在你的集群中
date: 2018/12/29 17:00:00
---

作者: [洪志国](https://github.com/honkiko)

## 前言
绝大多数的kubernetes集群都有这个隐患。只不过一般情况下，泄漏得比较慢，还没有表现出来而已。 

一个pod可能泄漏两个memory cgroup数量配额。即使pod百分之百发生泄漏， 那也需要一个节点销毁过三万多个pod之后，才会造成后续pod创建失败。

一旦表现出来，这个节点就彻底不可用了，必须重启才能恢复。

## 故障表现
腾讯云SCF(Serverless Cloud Function)底层使用我们的TKE(Tencent Kubernetes Engine)，并且会在节点上频繁创建和消耗容器。

SCF发现很多节点会出现类似以下报错，创建POD总是失败：

```
Dec 24 11:54:31 VM_16_11_centos dockerd[11419]: time="2018-12-24T11:54:31.195900301+08:00" level=error msg="Handler for POST /v1.31/containers/b98d4aea818bf9d1d1aa84079e1688cd9b4218e008c58a8ef6d6c3c106403e7b/start returned error: OCI runtime create failed: container_linux.go:348: starting container process caused \"process_linux.go:279: applying cgroup configuration for process caused \\\"mkdir /sys/fs/cgroup/memory/kubepods/burstable/pod79fe803c-072f-11e9-90ca-525400090c71/b98d4aea818bf9d1d1aa84079e1688cd9b4218e008c58a8ef6d6c3c106403e7b: no space left on device\\\"\": unknown"
```

这个时候，到节点上尝试创建几十个memory cgroup (以root权限执行 `for i in `seq 1 20`;do mkdir /sys/fs/cgroup/memory/${i}; done`)，就会碰到失败：

```
mkdir: cannot create directory '/sys/fs/cgroup/memory/8': No space left on device
```

其实，dockerd出现以上报错时， 手动创建***一个***memory cgroup都会失败的。 不过有时候随着一些POD的运行结束，可能会多出来一些“配额”，所以这里是尝试创建20个memory cgroup。


出现这样的故障以后，重启docker，释放内存等措施都没有效果，只有重启节点才能恢复。


## 复现条件
docker和kubernetes社区都有关于这个问题的issue:
- https://github.com/moby/moby/issues/29638
- https://github.com/kubernetes/kubernetes/issues/70324

网上有文章介绍了类似问题的分析和复现方法。如：
http://www.linuxfly.org/kubernetes-19-conflict-with-centos7/?from=groupmessage

不过按照文中的复现方法，我在`3.10.0-862.9.1.el7.x86_64`版本内核上并没有复现出来。

经过反复尝试，总结出了必现的复现条件。 一句话感慨就是，把进程加入到一个开启了kmem accounting的memory cgroup***并且执行fork系统调用***。

1. centos 3.10.0-862.9.1.el7.x86_64及以下内核， 4G以上空闲内存，root权限。
2. 把系统memory cgroup配额占满
    ```
    for i in `seq 1 65536`;do mkdir /sys/fs/cgroup/memory/${i}; done
    ```
    会看到报错：
    
    ```
    mkdir: cannot create directory ‘/sys/fs/cgroup/memory/65530’: No space left on device
    ```
    这是因为这个版本内核写死了，最多只能有65535个memory cgroup共存。 systemd已经创建了一些，所以这里创建不到65535个就会遇到报错。

    确认删掉一个memory cgroup, 就能腾出一个“配额”：

    ```
    rmdir /sys/fs/cgroup/memory/1
    mkdir /sys/fs/cgroup/memory/test
    ```
3. 给一个memory cgroup开启kmem accounting
    ```
    cd /sys/fs/cgroup/memory/test/
    echo 1 > memory.kmem.limit_in_bytes
    echo -1 > memory.kmem.limit_in_bytes
    ```

4. 把一个进程加进某个memory cgroup, 并执行一次fork系统调用
    
    ```
    最简单的就是把当前shell进程加进去：
    echo $$ > /sys/fs/cgroup/memory/test/tasks
    sleep 100 &
    cat /sys/fs/cgroup/memory/test/tasks
    ```

5. 把该memory cgroup里面的进程都挪走

    ```
    for p in `cat /sys/fs/cgroup/memory/test/tasks`;do echo ${p} > /sys/fs/cgroup/memory/tasks; done
    
    cat /sys/fs/cgroup/memory/test/tasks  //这时候应该为空
    ```

6. 删除这个memory cgroup

    ```
    rmdir /sys/fs/cgroup/memory/test
    ```

7. 验证刚才删除一个memory cgroup， 所占的配额并没有释放

    ```
    mkdir /sys/fs/cgroup/memory/xx
    ```
    这时候会报错：`mkdir: cannot create directory ‘/sys/fs/cgroup/memory/xx’: No space left on device`



## 什么版本的内核有这个问题
搜索内核commit记录，有一个commit应该是解决类似问题的：

```
4bdfc1c4a943: 2015-01-08 memcg: fix destination cgroup leak on task charges migration [Vladimir Davydov]
```

这个commit在3.19以及4.x版本的内核中都已经包含。 不过从docker和kubernetes相关issue里面的反馈来看，内核中应该还有其他cgroup泄漏的代码路径， 4.14版本内核都还有cgroup泄漏问题。


## 规避办法
不开启kmem accounting (以上复现步骤的第3步)的话，是不会发生cgroup泄漏的。

kubelet和runc都会给memory cgroup开启kmem accounting。所以要规避这个问题，就要保证kubelet和runc，都别开启kmem accounting。下面分别进行说明。 

## runc

查看代码，发现在commit fe898e7 (2017-2-25, PR #1350)以后的runc版本中，都会默认开启kmem accounting。代码在libcontainer/cgroups/fs/kmem.go: (老一点的版本，代码在libcontainer/cgroups/fs/memory.go)
```
const cgroupKernelMemoryLimit = "memory.kmem.limit_in_bytes"

func EnableKernelMemoryAccounting(path string) error {
        // Ensure that kernel memory is available in this kernel build. If it
        // isn't, we just ignore it because EnableKernelMemoryAccounting is
        // automatically called for all memory limits.
        if !cgroups.PathExists(filepath.Join(path, cgroupKernelMemoryLimit)) {
                return nil
        }
        // We have to limit the kernel memory here as it won't be accounted at all
        // until a limit is set on the cgroup and limit cannot be set once the
        // cgroup has children, or if there are already tasks in the cgroup.
        for _, i := range []int64{1, -1} {
                if err := setKernelMemory(path, i); err != nil {
                        return err
                }
        }
        return nil
}
```

runc社区也注意到这个问题，并做了比较灵活的修复： https://github.com/opencontainers/runc/pull/1921

这个修复给runc增加了"nokmem"编译选项。缺省的release版本没有打开这个选项，适用于4.x的高版本内核。 如果要在3.x版本内核上使用runc， 那就需要自己编译：

```
cd $GO_PATH/src/github.com/opencontainers/runc/
make BUILDTAGS="seccomp nokmem"
```

## kubelet
kubelet在创建pod对应的cgroup目录时，也会调用libcontianer中的代码对cgroup做设置。在   `pkg/kubelet/cm/cgroup_manager_linux.go`的Create方法中，会调用Manager.Apply方法，最终调用`vendor/github.com/opencontainers/runc/libcontainer/cgroups/fs/memory.go`中的MemoryGroup.Apply方法，开启kmem accounting。

这里也需要进行处理，可以不开启kmem accounting， 或者通过命令行参数来控制是否开启。

kubernetes社区也有issue讨论这个问题：https://github.com/kubernetes/kubernetes/issues/70324

但是目前还没有结论。我们TKE先直接把这部分代码注释掉了，不开启kmem accounting。




