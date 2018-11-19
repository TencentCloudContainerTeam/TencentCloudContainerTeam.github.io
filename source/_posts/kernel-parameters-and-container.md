
# sysctl

/proc/sys/目录下导出了一些可以在运行时修改kernel参数的proc文件。

```
# ls /proc/sys
abi  crypto  debug  dev  fs  kernel  net  vm
```

可以通过写proc文件来修改这些内核参数。例如， 要打开ipv4的路由转发功能：

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```
也可以通过sysctl命令来完成(只是对以上写proc文件操作的简单包装)：

```
sysctl -w net.ipv4.ip_forward=1
```

其他常用sysctl命令：

显示本机所有sysctl内核参数及当前值

```
sysctl -a
```

从文件(缺省使用/etc/sysctl.conf)加载多个参数和取值，并写入内核

```
sysctl -p [FILE]
```

另外， 系统启动的时候， 会自动执行一下"sysctl -p"。 所以，希望重启之后仍然生效的参数值， 应该写到/etc/sysctl.conf文件里面。


# 容器与sysctl
内核方面做了大量的工作，把一部分sysctl内核参数进行了namespace化(namespaced)。 也就是多个容器和主机可以各自独立设置某些内核参数。例如， 可以通过net.ipv4.ip_local_port_range，在不同容器中设置不同的端口范围。

如何判断一个参数是不是namespaced? 

运行一个具有privileged权限的容器(参考下一节内容)， 然后在容器中修改该参数，看一下在host上能否看到容器在中所做的修改。如果看不到， 那就是namespaced， 否则不是。


目前已经namespace化的sysctl内核参数：
- kernel.shm*,
- kernel.msg*,
- kernel.sem,
- fs.mqueue.*,
- net.*.

注意， vm.*并没有namespace化。 比如vm.max_map_count， 在主机或者一个容器中设置它， 其他所有容器都会受影响，都会看到最新的值。

# 在docker容器中修改sysctl内核参数
正常运行的docker容器中，是不能修改任何sysctl内核参数的。因为/proc/sys是以只读方式挂载到容器里面的。

```
proc on /proc/sys type proc (ro,nosuid,nodev,noexec,relatime)
```

要给容器设置不一样的sysctl内核参数，有多种方式。

### 方法一 --privileged

```
# docker run --privileged -it ubuntu bash
```

整个/proc目录都是以"rw"权限挂载的
```
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
```
在容器中，可以任意修改sysctl内核参赛。

注意： 
如果修改的是namespaced的参数， 则不会影响host和其他容器。反之，则会影响它们。

如果想在容器中修改主机的net.ipv4.ip_default_ttl参数， 则除了--privileged， 还需要加上 --net=host。

### 方法二 把/proc/sys bind到容器里面

```
# docker run -v /proc/sys:/writable-sys -it ubuntu bash
```
然后写bind到容器内的proc文件

```
echo 62 > /writable-sys/net/ipv4/ip_default_ttl
```
注意：  这样操作，效果类似于"--privileged", 对于namespaced的参数，不会影响host和其他容器。


### 方法三 --sysctl
```
# docker run -it --sysctl 'net.ipv4.ip_default_ttl=63' ubuntu sysctl net.ipv4.ip_default_ttl
net.ipv4.ip_default_ttl = 63
```
注意:
- 只有namespaced参数才可以。否则会报错"invalid argument..."
- 这种方式只是在容器初始化过程中完成内核参数的修改，容器运行起来以后，/proc/sys仍然是以只读方式挂载的，在容器中不能再次修改sysctl内核参数。



# kubernetes 与 sysctl
### 方法一 通过sysctls和unsafe-sysctls annotation
k8s还进一步把syctl参数分为safe和unsafe。 safe的条件：
- must not have any influence on any other pod on the node
- must not allow to harm the node’s health
- must not allow to gain CPU or memory resources outside of the resource limits of a pod.


非namespaced的参数，肯定是unsafe。

namespaced参数，也只有一部分被认为是safe的。

在pkg/kubelet/sysctl/whitelist.go中维护了safe sysctl参数的名单。在1.7.8的代码中，只有三个参数被认为是safe的：
- kernel.shm_rmid_forced,
- net.ipv4.ip_local_port_range,
- net.ipv4.tcp_syncookies

如果要设置一个POD中safe参数，通过security.alpha.kubernetes.io/sysctls这个annotation来传递给kubelet。

```
metadata:
  name: sysctl-example
  annotations:
    security.alpha.kubernetes.io/sysctls: kernel.shm_rmid_forced=1
```


如果要设置一个namespaced， 但是unsafe的参数，要使用另一个annotation: security.alpha.kubernetes.io/unsafe-sysctls， 另外还要给kubelet一个特殊的启动参数。


```
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-example
  annotations:
    security.alpha.kubernetes.io/sysctls: kernel.shm_rmid_forced=1
    security.alpha.kubernetes.io/unsafe-sysctls: net.ipv4.route.min_pmtu=1000,kernel.msgmax=1 2 3
spec:
  ...
```

kubelet 增加--experimental-allowed-unsafe-sysctls启动参数

```
kubelet --experimental-allowed-unsafe-sysctls 'kernel.msg*,net.ipv4.route.min_pmtu'
```

### 方法二 privileged POD

如果要修改的是非namespaced的参数， 如vm.*， 那就没办法使用以上方法。 可以给POD privileged权限，然后在容器的初始化脚本或代码中去修改sysctl参数。

创建POD/deployment/daemonset等对象时， 给容器的spec指定securityContext.privileged=true
```
    spec:
      containers:
      - image: nginx:alpine
        securityContext:
          privileged: true
```
这样跟"docker run --privileged"效果一样，在POD中/proc是以"rw"权限mount的，可以直接修改相关sysctl内核参数。



# ulimit
每个进程都有若干操作系统资源的限制， 可以通过 /proc/$PID/limits 来查看。

```
$ cat /proc/1/limits 
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             62394                62394                processes 
Max open files            1024                 4096                 files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       62394                62394                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us
```

在bash中有个ulimit内部命令，可以查看当前bash进程的这些限制。

跟ulimit属性相关的配置文件是/etc/security/limits.conf。具体配置项和语法可以通过`man limits.conf` 命令查看。


### systemd给docker daemon自身配置ulimit
在service文件中(一般是/usr/lib/systemd/system/dockerd.service)中可以配置：

```
[Service]
LimitAS=infinity
LimitRSS=infinity
LimitCORE=infinity
LimitNOFILE=65536
ExecStart=...
WorkingDirectory=...
User=...
Group=...
```


### dockerd 给容器的 缺省ulimit设置
dockerd --default-ulimit nofile=65536:65536

冒号前面是soft limit, 后面是hard limit

### 给容器指定ulimit设置
docker run -d --ulimit nofile=20480:40960 nproc=1024:2048 容器名

### 在kubernetes中给pod设置ulimit参数
有一个issue在讨论这个问题： https://github.com/kubernetes/kubernetes/issues/3595

目前可行的办法，是在镜像中的初始化程序中调用setrlimit()系统调用来进行设置。子进程会继承父进程的ulimit参数。


# 参考文档：
http://tapd.oa.com/CCCM/prong/stories/view/1010166561060564549

https://kubernetes.io/docs/concepts/cluster-administration/sysctl-cluster/

https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities



