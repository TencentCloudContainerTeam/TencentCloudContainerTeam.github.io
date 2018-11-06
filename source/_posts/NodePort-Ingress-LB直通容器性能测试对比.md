---
title: NodePort, Ingress, LB直通Pod性能测试对比
date: 2018/11/06 15:45:37
---

作者：郭志宏 


### 1. 测试背景：

目前基于k8s 服务的外网访问方式有以下几种：

1. NodePort  
2. Ingress (如社区版的nginx ingress)
3. 自研 LB -> Pod （比如pod ip 作为 nginx 的 upstream）

其中第一种和第二种方案都要经过iptables 转发，第三种方案不经过iptables，本测试主要是为了测试这三种方案的性能损耗。



### 2. 测试方案

为了做到测试的准确性和全面性，我们提供以下测试工具和测试数据：

1. 2核4G 的Pod

2. 5个Node 的4核8G 集群

3. 16核32G 的Nginx 作为统一的LB

4. 一个测试应用，2个静态测试接口，分别对用不同大小的数据包（4k 和 100K）

5. 测试1个pod ，10个pod的情况（service/pod 越多，一个机器上的iptables 规则数就越多，关于iptables规则数对转发性能的影响，在“ipvs和iptables模式下性能对⽐比测试报告” 已有结论： Iptables场景下，对应service在总数为2000内时，每个service 两个pod, 性能没有明显下降。当service总数达到3000、4000时，性能下降明显，service个数越多，性能越差。）所以这里就不考虑pod数太多的情况。

6. 单独的16核32G 机器作作为压力机，使用wrk 作为压测工具, qps 作为评估标准，

7. 那么每种访问方式对应以下4种情况


   | 测试用例 | Pod 数 | 数据包大小 | 平均QPS |
   | ---- | ----- | ----- | ----- |
   | 1    | 1     | 4k    |       |
   | 2    | 1     | 100K  |       |
   | 3    | 10    | 4k    |       |
   | 4    | 10    | 100k  |       |

8. 每种情况测试5次，取平均值（qps），完善上表。

### 3. 测试过程

1. 准备一个测试应用（基于nginx），提供两个静态文件接口，分别返回4k的数据和100K 的数据。

   镜像地址：ccr.ccs.tencentyun.com/caryguo/nginx:v0.1

   接口：http://0.0.0.0/4k.html

   ​            http://0.0.0.0/100k.htm

2. 部署压测工具。https://github.com/wg/wrk

3. 部署集群，5台Node来调度测试Pod, 10.0.4.6 这台用来独部署Nginx, 作为统一的LB, 将这台机器加入集群的目的是为了 将ClusterIP 作为nginx 的upstream .

   ```
   root@VM-4-6-ubuntu:/etc/nginx# kubectl get node
   NAME        STATUS                     ROLES     AGE       VERSION
   10.0.4.12   Ready                      <none>    3d        v1.10.5-qcloud-rev1
   10.0.4.3    Ready                      <none>    3d        v1.10.5-qcloud-rev1
   10.0.4.5    Ready                      <none>    3d        v1.10.5-qcloud-rev1
   10.0.4.6    Ready,SchedulingDisabled   <none>    12m       v1.10.5-qcloud-rev1
   10.0.4.7    Ready                      <none>    3d        v1.10.5-qcloud-rev1
   10.0.4.9    Ready                      <none>    3d        v1.10.5-qcloud-rev1
   ```

4. 根据不同的测试场景，调整Nginx 的upstream, 根据不同的Pod, 调整压力，让请求的超时率控制在万分之一以内,  数据如下：

   ```
   ./wrk -c 200 -d 20 -t 10 http://carytest.pod.com/10k.html   单pod
   ./wrk -c 1000 -d 20 -t 100 http://carytest.pod.com/4k.html  10 pod
   ```

5. 测试wrk -> nginx -> Pod 场景，

| 测试用例 | Pod 数 | 数据包大小 | 平均QPS |
| ---- | ----- | ----- | ----- |
| 1    | 1     | 4k    | 12498 |
| 2    | 1     | 100K  | 2037  |
| 3    | 10    | 4k    | 82752 |
| 4    | 10    | 100k  | 7743  |

5. 模拟测试Nginx ingress 场景。 wrk -> nginx -> ClusterIP -> Pod

| 测试用例 | Pod 数 | 数据包大小 | 平均QPS |
| ---- | ----- | ----- | ----- |
| 1    | 1     | 4k    | 12568 |
| 2    | 1     | 100K  | 2040  |
| 3    | 10    | 4k    | 81752 |
| 4    | 10    | 100k  | 7824  |

6. NodePort 场景，wrk -> nginx -> NodePort -> Pod

| 测试用例 | Pod 数 | 数据包大小 | 平均QPS |
| ---- | ----- | ----- | ----- |
| 1    | 1     | 4k    | 12332 |
| 2    | 1     | 100K  | 2028  |
| 3    | 10    | 4k    | 76973 |
| 4    | 10    | 100k  | 5676  |



压测过程中，4k 数据包的情况下，应用的负载都在80% -100% 之间， 100k 情况下，应用的负载都在20%-30%

之间，压力都在网络消耗上，没有到达服务后端。

### 4. 测试结论

1. 在一个pod 的情况下（4k 或者100 数据包），3中网络方案差别不大，QPS 差距在3% 以内。
2. 在10个pod，4k 数据包情况下，lb->pod 和 nginx ingress 差距不大，NodePort 损失近7% 左右。
3. 10个Pod, 100k 数据包的情况下，lb->pod 和 nginx ingress 差距不大，NodePort 损失近 25% 

### 5. 附录

1. nginx 配置

```
user nginx;
worker_processes 50;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 100000;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;
    
     # pod ip
    upstream  panda-pod {
          #ip_hash;
          # Pod ip
          #server   10.0.4.12:30734  max_fails=2 fail_timeout=30s;
          #server   172.16.1.5:80  max_fails=2 fail_timeout=30s;
          #server   172.16.2.3:80  max_fails=2 fail_timeout=30s;
          #server   172.16.3.5:80  max_fails=2 fail_timeout=30s;
          #server   172.16.4.6:80  max_fails=2 fail_timeout=30s;
          #server   172.16.4.5:80  max_fails=2 fail_timeout=30s;
          #server   172.16.3.6:80  max_fails=2 fail_timeout=30s;
          #server   172.16.1.4:80  max_fails=2 fail_timeout=30s;
          #server   172.16.0.7:80  max_fails=2 fail_timeout=30s;
          #server   172.16.0.6:80  max_fails=2 fail_timeout=30s;
          #server   172.16.2.2:80  max_fails=2 fail_timeout=30s;
          
          # svc ip
          #server   172.16.255.121:80  max_fails=2 fail_timeout=30s;
           
          # NodePort
          server   10.0.4.12:30734   max_fails=2 fail_timeout=30s;
          server   10.0.4.3:30734    max_fails=2 fail_timeout=30s;
          server   10.0.4.5:30734    max_fails=2 fail_timeout=30s;
          server   10.0.4.7:30734    max_fails=2 fail_timeout=30s;
          server   10.0.4.9:30734    max_fails=2 fail_timeout=30s;
		    
          keepalive 256;
    }

    server {
        listen       80;
        server_name  carytest.pod.com;
        # root   /usr/share/nginx/html;
        charset utf-8;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
        location / {
                    proxy_pass        http://panda-pod;
                    proxy_http_version 1.1;
                    proxy_set_header Connection "";
                    proxy_redirect off;
                    proxy_set_header  Host  $host;
                    proxy_set_header  X-Real-IP  $remote_addr;
                    proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
                    proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;

        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
        
```









