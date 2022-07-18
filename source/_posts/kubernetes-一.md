---
title: kubernetes(一)
categories: kubernetes
tags:
  - 运维
  - minikube
abbrlink: e2edcca7
date: 2022-02-20 19:45:44
---

# kubernetes 概念

- 相关参考文档
  - https://k8s.easydoc.net/docs/dRiQjyTY/28366845/6GiNOzyZ/nd7yOvdY
  - https://kubernetes.io/zh/docs/tasks/
- 什么是 kubernetes ?
  - 为 容器化  应用提供集群部署和管理的开源工具，特点是 声明式，自动化
  - 运维部署的发展，
    - 物理机部署；
    - 虚拟化部署（对物理机进行抽象=>多个虚拟机（模拟完整的计算机））
    - 容器化部署（以应用为基础进行抽象=>给每一个应用模拟一个完整的计算机环境，实际上是共享资源，屏蔽了底层的OS，可移植性更强，资源利用率更高，更适合微服务开发运维）
  - 特性 （可弹性运行分布式系统的框架）
    - 服务发现和负载均衡
    - 存储编排
    - 自动部署和回滚
    - 自我修复
    - 密钥与配置管理
  
- 什么时候需要 kubernetes ？
  - 前提条件 - 应用容器化
  - 单机， docker + docker-compose  就足够了
  - 3，4 台机器，shell 脚本 +负载均衡 也够用了
  - 需要部署 更多台机器时，运维工作会很痛苦
  - kubernetes 最多支持 5000  node(类似于 5000 台机器)
    - 每个 node  最默认最大支持 Pod (类似于 application ) 110
    - pod  总数 不超过 15W
    - 容器总数不超过 30W


## kubernetes 的组成

###   Node 节点(部署了 kubernetes 的机器)
- 主节点 Master （控制与调度）
- 工作节点 Worker
  - Pod （k8s 调度的最小单元，可以包含一个或多个容器）
       - 虚拟 IP    

### 组件

#### 控制平面组件 （Control Plane Components）

   对集群进行调度&检测
- kube-apiserver  (kubernetes api 服务器 是控制面的前端)
- etcd (存储集群信息的后端键值对数据库)
- kube-scheduler (监视新创建、未执行运行节点的 Pods, 选择节点 让 Pod 在上面运行)
- kube-controller-manager (运行控制器的组件)
  -  节点控制器
  - 任务控制器
  - 端点控制器
  - 账户服务和令牌控制器
-  cloud-controller-manager (云控制器，可以将集群连接到云服务商的API上)
   
#### Node 组件
  在每个节点上运行，维护运行的Pod 提供 kubernetes 运行环境
- kubelet (运行在node 上的代理，保证 容器都运行在 Pod 中, 只控制 由 kubernetes 创建的容器)
- kebe-proxy  (负责节点的网络代理)
-  容器运行时 （Container Runtime, 符合CRI 的环境 如 docker）


#### 插件 Addons ( 提供集群级别功能)

- DNS
- Web 界面（仪表盘）
- 容器资源监控
- 集群层面日志

##  installer

- minikube
- 云平台kebernetes
- 裸机安装




### docker 的安装
```shell
# 将防火墙 配置全部置空

systemctl stop firewalld && systemctl disable firewalld

iptables -F
iptables -X
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT

# 关闭内存交换
swapoff -a

# 关闭 selinux 
vim /etc/selinux/config
# 修改后重启
SELINUX=disabled

yum remove docker  docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

yum install -y yum-utils

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install docker-ce docker-ce-cli containerd.io -y

systemctl start docker

systemctl enable docker
```

### minikube 的安装

#### 安装 kubectl

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubectl

# 或者直接拉二进制文件
curl -Lo kubectl https://kubernetes.oss-cn-hangzhou.aliyuncs.com/kubernetes-release/release/v1.20.2/bin/linux/amd64/kubectl && chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

#### 安装 单机 minikube

```shell
curl -Lo minikube https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/release/v1.20.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

yum install epel-release -y 
yum install conntrack-tools -y

minikube version

sudo /usr/local/bin/minikube start --registry-mirror=https://registry.docker-cn.com --vm-driver=none --alsologtostderr -v=4
```

安装 k8s 面板

```shell
yum install xdg-utils 

minikube dashboard

# 出现下列信息 就已经开启
* 正在验证 dashboard 运行情况 ...
* Launching proxy ...
* 正在验证 proxy 运行状况 ...
http://127.0.0.1:46491/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```

##### 可能会碰到的问题 

-  minikube 卡在 正在验证 proxy 运行状况 ... 

```shell
systemctl stop kubectl
systemctl stop docker

# 将防火墙 配置全部置空
iptables -F
iptables -X
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT

systemctl start kubectl
systemctl start docker
```
- 本地策略限制本机访问

```shell
# 思路一 ，本机开 nginx 代理转发流量
upstream api_k8s_service {
  server localhost:46491;
}

server {
    listen       8099;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
	proxy_set_header Host 127.0.0.1;
        proxy_pass http://api_k8s_service/;
    }

}

# 思路二 
kubectl proxy --address='0.0.0.0' --accept-hosts='^*$'

# 这样就可以直接访问 不需要 minikuba dashboard
http://xx.xx.x.xx:8001/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/overview?namespace=default
```

- 访问 dashboard 异常

```shell
# 访问
http://xx.xx.x.xx:8001/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/overview?namespace=default

```
- 排查思路 先确认是 哪一个 pod 有问题
  - kubectl get pods --namespace kube-system
- 查看 READY 状态不正常 的pod log
  - kubectl describe pod  kube-proxy-h57wx --namespace kube-system

## 应用部署
