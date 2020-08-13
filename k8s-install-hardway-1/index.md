# 二进制部署Kubernetes Part1


## 前言

### 项目地址

- 本次安装完全参照于 [follow-me-install-kubernetes-cluster](https://github.com/opsnull/follow-me-install-kubernetes-cluster/) 项目

- 对于源项目操作进行了部分扩展的解释与优化

### 版本信息

- 当前版本基于 v1.16.x 版本



## 环境准备

### 版本信息

| 组件                   | 版本       | 发布时间   |
| ---------------------- | ---------- | ---------- |
| kubernetes             | 1.16.6     | 2020-01-22 |
| etcd                   | 3.4.3      | 2019-10-24 |
| containerd             | 1.3.3      | 2020-02-07 |
| runc                   | 1.0.0-rc10 | 2019-12-23 |
| calico                 | 3.12.0     | 2020-01-27 |
| coredns                | 1.6.6      | 2019-12-20 |
| dashboard              | v2.0.0-rc4 | 2020-02-06 |
| k8s-prometheus-adapter | 0.5.0      | 2019-04-03 |
| prometheus-operator    | 0.35.0     | 2020-01-13 |
| prometheus             | 2.15.2     | 2020-01-06 |
| elasticsearch、kibana  | 7.2.0      | 2019-06-25 |
| cni-plugins            | 0.8.5      | 2019-12-20 |
| metrics-server         | 0.3.6      | 2019-10-15 |



## 主要配置策略

kube-apiserver：

- 使用节点本地 nginx 4 层透明代理实现高可用；
- 关闭非安全端口 8080 和匿名访问；
- 在安全端口 6443 接收 https 请求；
- 严格的认证和授权策略 (x509、token、RBAC)；
- 开启 bootstrap token 认证，支持 kubelet TLS bootstrapping；
- 使用 https 访问 kubelet、etcd，加密通信；

kube-controller-manager：

- 3 节点高可用；
- 关闭非安全端口，在安全端口 10252 接收 https 请求；
- 使用 kubeconfig 访问 apiserver 的安全端口；
- 自动 approve kubelet 证书签名请求 (CSR)，证书过期后自动轮转；
- 各 controller 使用自己的 ServiceAccount 访问 apiserver；

kube-scheduler：

- 3 节点高可用；
- 使用 kubeconfig 访问 apiserver 的安全端口；

kubelet：

- 使用 kubeadm 动态创建 bootstrap token，而不是在 apiserver 中静态配置；
- 使用 TLS bootstrap 机制自动生成 client 和 server 证书，过期后自动轮转；
- 在 KubeletConfiguration 类型的 JSON 文件配置主要参数；
- 关闭只读端口，在安全端口 10250 接收 https 请求，对请求进行认证和授权，拒绝匿名访问和非授权访问；
- 使用 kubeconfig 访问 apiserver 的安全端口；

kube-proxy：

- 使用 kubeconfig 访问 apiserver 的安全端口；
- 在 KubeProxyConfiguration 类型的 JSON 文件配置主要参数；
- 使用 ipvs 代理模式；

集群插件：

- DNS：使用功能、性能更好的 coredns；
- Dashboard：支持登录认证；
- Metric：metrics-server，使用 https 访问 kubelet 安全端口；
- Log：Elasticsearch、Fluend、Kibana；
- Registry 镜像库：docker-registry、harbor；



### 机器信息

| IP            | 主机名     | 备注 |
| ------------- | ---------- | ---- |
| 192.168.7.200 | k8s-node01 |      |
| 192.168.7.201 | k8s-node02 |      |
| 192.168.7.203 | k8s-node03 |      |



## 基础配置

### 注意事项

- 基础配置章节，如果没有特别指明在 k8s-node01 节点，所有操作所有节点都需要进行



### 主机名

- 修改机器主机名

```shell
hostnamectl set-hostname k8s-node01
hostnamectl set-hostname k8s-node02
hostnamectl set-hostname k8s-node03
```

- 修改 hosts 信息，关联节点DNS解析

```shell
cat	>>	/etc/hosts <<EOF
	192.168.7.200 k8s-node01
	192.168.7.201 k8s-node02
	192.168.7.202 k8s-node03
EOF
```

- 重新连接当前 SSH 连接，主机名显示生效



### 节点免密

- **在 k8s-node01节点执行**
- 保证 k8s-node01 可以免密登录到其他机器

```shell
ssh-keygen -t rsa
ssh-copy-id root@k8s-node01
ssh-copy-id root@k8s-node02
ssh-copy-id root@k8s-node03
```



### 添加本次二进制程序存放路径到Path

- 以 /opt/k8s 作为部署根目录，二进制文件存放在 bin 中，添加到 Path
- 具体目录后续会创建

```shell
echo  'PATH=/opt/k8s/bin:$PATH'	>>/root/.bashrc 
source	/root/.bashrc
```



### 依赖安装

- kube-proxy 使用 ipvs 模式安装， ipvsadm 为 ipvs 的管理工具
- etcd 集群节点之间需要时间同步，chrony 用于系统时间同步

```shell
yum install -y epel-release 
yum install -y chrony conntrack ipvsadm ipset jq iptables curl sysstat libseccomp wget socat git
```



### 防火墙配置

- 关闭防火墙
- 清空 iptables 规则
- 设置默认转发策略

```shell
systemctl stop firewalld && systemctl disable firewalld 
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
iptables -P FORWARD ACCEPT
```



### 关闭SWAP分区

- 关闭swap分区，否则可能会影响 kubelet 启动
- 同样可以通过设置 kubelet 启动参数 --fail-swap-on 为 false 关闭 swap 检查

```shell
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```



### 关闭 SELINUX

- 关闭 SELINUX 否则可能 kubelet 挂载目录报错：Permission denied

```shell
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```



### 内核调优

- 关闭 tcp_tw_recycle，否则与 NAT 冲突， 可能导致服务不通；

```shell
cat	> /etc/sysctl.d/kubernetes.conf	<<EOF
    net.bridge.bridge-nf-call-iptables=1
    net.bridge.bridge-nf-call-ip6tables=1
    net.ipv4.ip_forward=1
    net.ipv4.tcp_tw_recycle=0
    net.ipv4.neigh.default.gc_thresh1=1024
    net.ipv4.neigh.default.gc_thresh1=2048
    net.ipv4.neigh.default.gc_thresh1=4096
    vm.swappiness=0 vm.overcommit_memory=1
    vm.panic_on_oom=0
    fs.inotify.max_user_instances=8192
    fs.inotify.max_user_watches=1048576
    fs.file-max=52706963 fs.nr_open=52706963
    net.ipv6.conf.all.disable_ipv6=1
    net.netfilter.nf_conntrack_max=2310720
EOF
```

- 否则会执行 调优参数 生效的时候报错：sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory

```shell
modprobe br_netfilter
```

```shell
sysctl -p /etc/sysctl.d/kubernetes.conf
```



### 设置时区以及同步

```shell
timedatectl set-timezone Asia/Shanghai
systemctl enable chronyd && systemctl start chronyd
```

- 检查状态

```shell
timedatectl status
```

- 如果输出内容存在 即可

```shell
Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
```

- 将当前 UTC 时间写入硬件时钟，并重启系统时间相关的依赖服务

```shell
timedatectl set-local-rtc 0
systemctl restart rsyslog
systemctl restart crond
```



### 关闭无关服务

```shell
systemctl disable postfix && systemctl stop postfix
```



### 创建相关目录

```shell
mkdir -p /opt/k8s/{bin,work} /etc/{kubernetes,etcd}/cert
```



### 分发集群配置参数脚本

- **该操作在k8s-node1节点进行**
- 集群配置参数脚本， 根据具体情况更改机器信息   [脚本原地址](https://github.com/opsnull/follow-me-install-kubernetes-cluster/blob/master/manifests/environment.sh)

```shell
vim environment.sh
```

```shell
#!/usr/bin/bash

# 生成 EncryptionConfig 所需的加密 key
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

# 集群各机器 IP 数组
export NODE_IPS=(192.168.7.200 192.168.7.201 192.168.7.202)

# 集群各 IP 对应的主机名数组
export NODE_NAMES=(k8s-node01 k8s-node02 k8s-node03)

# etcd 集群服务地址列表
export ETCD_ENDPOINTS="https://192.168.7.200:2379,https://192.168.7.201:2379,https://192.168.7.202:2379"

# etcd 集群间通信的 IP 和端口
export ETCD_NODES="k8s-node01=https://192.168.7.201:2380,k8s-node02=https://192.168.7.201:2380,k8s-node03=https://192.168.7.202:2380"

# kube-apiserver 的反向代理(kube-nginx)地址端口
export KUBE_APISERVER="https://127.0.0.1:8443"

# 节点间互联网络接口名称
export IFACE="eth0"

# etcd 数据目录
export ETCD_DATA_DIR="/data/k8s/etcd/data"

# etcd WAL 目录，建议是 SSD 磁盘分区，或者和 ETCD_DATA_DIR 不同的磁盘分区
export ETCD_WAL_DIR="/data/k8s/etcd/wal"

# k8s 各组件数据目录
export K8S_DIR="/data/k8s/k8s"

## DOCKER_DIR 和 CONTAINERD_DIR 二选一
# docker 数据目录
export DOCKER_DIR="/data/k8s/docker"

# containerd 数据目录
export CONTAINERD_DIR="/data/k8s/containerd"

## 以下参数一般不需要修改

# TLS Bootstrapping 使用的 Token，可以使用命令 head -c 16 /dev/urandom | od -An -t x | tr -d ' ' 生成
BOOTSTRAP_TOKEN="41f7e4ba8b7be874fcff18bf5cf41a7c"

# 最好使用 当前未用的网段 来定义服务网段和 Pod 网段

# 服务网段，部署前路由不可达，部署后集群内路由可达(kube-proxy 保证)
SERVICE_CIDR="10.254.0.0/16"

# Pod 网段，建议 /16 段地址，部署前路由不可达，部署后集群内路由可达(flanneld 保证)
CLUSTER_CIDR="172.30.0.0/16"

# 服务端口范围 (NodePort Range)
export NODE_PORT_RANGE="30000-32767"

# kubernetes 服务 IP (一般是 SERVICE_CIDR 中第一个IP)
export CLUSTER_KUBERNETES_SVC_IP="10.254.0.1"

# 集群 DNS 服务 IP (从 SERVICE_CIDR 中预分配)
export CLUSTER_DNS_SVC_IP="10.254.0.2"

# 集群 DNS 域名（末尾不带点号）
export CLUSTER_DNS_DOMAIN="cluster.local"

# 将二进制目录 /opt/k8s/bin 加到 PATH 中
export PATH=/opt/k8s/bin:$PATH
```



- 分发到各个节点

```shell
source environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    scp environment.sh root@${node_ip}:/opt/k8s/bin/
    ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
  done
```



### 升级内核

CentOS 7.x 内核 自带的 3.10.x  存在一些 Bugs

- 高版本的 docker(1.13之后) 启用了 3.10 kernel 实验室支持的 kernel memory account 功能（无法关闭），会在节点压力大（比如频繁启动关闭容器） 时候导致  cgroup memory leak
- 网络设备引用计数泄露，会导致类似于报错："kernel:unregister_netdevice: waiting for eth0 to become free. Usage count = 1";



采用升级内核至 4.4.x 以上的方法

```shell
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel install -y kernel-lt
```

- 检查 /boot/grub2/grub.cfg 中对应内核 menuentry 中是否包含 initrd16 配置, 如果无输出，再尝试安装一次 

- 设置开机启动内核

```shell
grub2-set-default 0
```

- 重启机器

```shell
sync
reboot
```

- 查看内核 是否已经升级为  4.4.x版本

```shell
uname -a
```





## 根证书创建



需要使用 X509 证书对通信进行加密和认证； 使用 CloudFlare 的PKI工具集 cfssl 生成，CA证书只需要一份

- 本小节所有操作均在 k8s-node01 节点下执行

### cfssl 工具集

```shell
mkdir -p /opt/k8s/cert && cd /opt/k8s/work

wget https://github.com/cloudflare/cfssl/releases/download/v1.4.1/cfssl_1.4.1_linux_amd64
mv cfssl_1.4.1_linux_amd64 /opt/k8s/bin/cfssl

wget https://github.com/cloudflare/cfssl/releases/download/v1.4.1/cfssljson_1.4.1_linux_amd64
mv cfssljson_1.4.1_linux_amd64 /opt/k8s/bin/cfssljson

wget https://github.com/cloudflare/cfssl/releases/download/v1.4.1/cfssl-certinfo_1.4.1_linux_amd64
mv cfssl-certinfo_1.4.1_linux_amd64 /opt/k8s/bin/cfssl-certinfo

chmod +x /opt/k8s/bin/*
```



### 创建配置文件

- CA 配置文件用于配置根证书的使用场景（profile）和 具体参数（usage，过期时间，服务端认证，客户端认证，加密等）

```shell
cd /opt/k8s/work

cat > ca-config.json << EOF
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "kubernetes": {
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ],
                "expiry": "876000h"
            }
        }
    }
}
EOF
```



| 字段        | 含义                                                 |
| ----------- | ---------------------------------------------------- |
| signing     | 表示该证书可以签名其他证书（ca.pem 中 CA=TRUE）      |
| server auth | 表示 client 可以用该证书对 server 提供的证书进行验证 |
| client auth | 表示 server 可以用该证书对 client 提供的证书进行验证 |
| expiry      | 标识证书的有效期，上述表示100年                      |



### 创建证书签名请求文件

```shell
cd /opt/k8s/work 
cat > ca-csr.json <<EOF
{
    "CN": "kubernetes-ca",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "opsnull"
        }
    ],
    "ca": {
        "expiry": "876000h"
    }
}
EOF
```

| 字段 | 含义                                                         |
| ---- | ------------------------------------------------------------ |
| CN   | Common Name kube-apiserver 从证书中提取该字段作为**请求的用户名（User Name）**；浏览器通过该字段验证网站是否合法 |
| O    | Organization：kube-apiserver从证书中提取该字段作为**请求用户所属的组 （Group）** |
|      | kube-apiserver 将**提取的 User、Group 作为 RBAC 授权的用户标识** |

注意：

- 不同证书的 csr 文件中的 CN, C, ST, L, O, OU 组合 必须不同，否则可能出现 PEER'S CERTIFICATE HAS AN INVAILD SINGTURE 错误；
- 后续创建的证书，保证CN不同以达到区分



### 生成 CA证书以及密钥

```shell
cd /opt/k8s/work
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
ls ca*
```



### 分发证书

```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh 
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "mkdir -p /etc/kubernetes/cert"
    scp ca*.pem ca-config.json root@${node_ip}:/etc/kubernetes/cert
  done
```



### 参考

- [Kubernetes证书类型](https://github.com/kubernetes-sigs/apiserver-builder-alpha/blob/master/docs/concepts/auth.md)
















