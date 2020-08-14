# 二进制部署Kubernetes Part2


## 前言

### 系列目录

- [二进制部署Kubernetes Part1](https://www.viper.pub/k8s-install-hardway-1/)
- [二进制部署Kubernetes Part2(本篇)](https://www.viper.pub/k8s-install-hardway-2/)
- [二进制部署Kubernetes Part3](https://www.viper.pub/k8s-install-hardway-3/)
- [二进制部署Kubernetes Part4](https://www.viper.pub/k8s-install-hardway-4/)



### 注意

- 本章节所有操作默认均在 `k8s-node01`下操作
- `kubeconfig` 文件通用， 存放一般在需要使用kubectl机器的 `~/.kube/config` 位置
- kubeconfig 是与 APIServer 交互的凭证，基于 `client-go` 对 Kubernetes 的二次开发也是通过该配置文件接入



## 部署配置 kubectl



### 下载分发二进制文件

```shell
cd /opt/k8s/work
wget https://dl.k8s.io/v1.16.6/kubernetes-client-linux-amd64.tar.gz
tar -xzvf kubernetes-client-linux-amd64.tar.gz
```

- 分发至各个节点，**如果只需要在主节点使用，可以不用分发**

```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    scp kubernetes/client/bin/kubectl root@${node_ip}:/opt/k8s/bin/
    ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
  done
```



### 创建 admin证书和密钥

kubectl 通过 `https` 与 kube-apiserver 进行安全通信;` kube-apiserver `对于 kubectl 请求包含的证书进行认证和授权

kubectl 用于整个集群的管理，所以创建**最高权限的 admin** 证书

```shell
cd /opt/k8s/work
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "opsnull"
    }
  ]
}
EOF
```

- `O: system:masters`：kube-apiserver 收到使用该证书的客户端请求后，为**请求添加组（Group）认证标识** `system:masters`；
- 预定义的 ClusterRoleBinding `cluster-admin` 将 Group `system:masters` 与 Role `cluster-admin` 绑定，该 Role 授予操作集群所需的**最高**权限；
- 该证书只会被 kubectl 当做 client 证书使用，所以 `hosts` 字段为空；

生成证书

```shell
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
  -ca-key=/opt/k8s/work/ca-key.pem \
  -config=/opt/k8s/work/ca-config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

- 忽略警告消息 `[WARNING] This certificate lacks a "hosts" field.`；



### 创建并分发 kubeconfig 文件

kubeconfig 包含 `kube-apiserver` 地址与认证信息，

```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/k8s/work/ca.pem \
  --embed-certs=true \
  --server=https://${NODE_IPS[0]}:6443 \
  --kubeconfig=kubectl.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials admin \
  --client-certificate=/opt/k8s/work/admin.pem \
  --client-key=/opt/k8s/work/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=kubectl.kubeconfig

# 设置上下文参数
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin \
  --kubeconfig=kubectl.kubeconfig

# 设置默认上下文
kubectl config use-context kubernetes --kubeconfig=kubectl.kubeconfig
```

- `--certificate-authority`：验证 kube-apiserver 证书的根证书；
- `--client-certificate`、`--client-key`：刚生成的 `admin` 证书和私钥，与 kube-apiserver https 通信时使用；
- `--embed-certs=true`：将 ca.pem 和 admin.pem 证书内容嵌入到生成的 kubectl.kubeconfig 文件中(否则，写入的是证书文件路径，后续拷贝 kubeconfig 到其它机器时，还需要单独拷贝证书文件，不方便。)；
- `--server`：指定 kube-apiserver 的地址，这里指向第一个节点上的服务；



分发到需要使用 `kubectl` 的节点， 如果只需要主节点，则不需要

- 分发配置文件到在主机的 `~/.kube/config` 下

```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "mkdir -p ~/.kube"
    scp kubectl.kubeconfig root@${node_ip}:~/.kube/config
  done
```



## 部署 etcd 集群

本文档构建三个节点高可用的 etcd 集群

注意：

- `flanneld` 插件与 `etcd v3.4.x` 版本不兼容，如果需要部署 flanneld 插件需要使用 `etcd v3.3.x` 



### 下载分发 etcd 二进制文件

```shell
cd /opt/k8s/work
wget https://github.com/coreos/etcd/releases/download/v3.4.3/etcd-v3.4.3-linux-amd64.tar.gz
tar -xvf etcd-v3.4.3-linux-amd64.tar.gz
```



```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    scp etcd-v3.4.3-linux-amd64/etcd* root@${node_ip}:/opt/k8s/bin
    ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
  done
```



### 创建 etcd 证书和私钥

- 以 `k8s` User Name 创建证书签名请求
- `hosts` 中需要包含 etcd 集群中所有成员地址

```shell
cd /opt/k8s/work
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.7.200",
    "192.168.7.201",
    "192.168.7.202"
  ],
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
  ]
}
EOF
```

- 生成证书公钥以及私钥

```shell
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
    -ca-key=/opt/k8s/work/ca-key.pem \
    -config=/opt/k8s/work/ca-config.json \
    -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
ls etcd*pem
```

- 分发到集群各个节点

```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "mkdir -p /etc/etcd/cert"
    scp etcd*.pem root@${node_ip}:/etc/etcd/cert/
  done
```



### 创建 etcd 的 systemd unit 模板文件



- 注意：此处对于模板文件的生成  ${}  引用环境变量**会直接输出值到文件**
- 如果环境变量值存在错误，模板文件会存在问题，需要重新回到此步生成模板文件

```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > etcd.service.template <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=${ETCD_DATA_DIR}
ExecStart=/opt/k8s/bin/etcd \\
  --data-dir=${ETCD_DATA_DIR} \\
  --wal-dir=${ETCD_WAL_DIR} \\
  --name=##NODE_NAME## \\
  --cert-file=/etc/etcd/cert/etcd.pem \\
  --key-file=/etc/etcd/cert/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-cert-file=/etc/etcd/cert/etcd.pem \\
  --peer-key-file=/etc/etcd/cert/etcd-key.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls=https://##NODE_IP##:2380 \\
  --initial-advertise-peer-urls=https://##NODE_IP##:2380 \\
  --listen-client-urls=https://##NODE_IP##:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://##NODE_IP##:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --auto-compaction-mode=periodic \\
  --auto-compaction-retention=1 \\
  --max-request-bytes=33554432 \\
  --quota-backend-bytes=6442450944 \\
  --heartbeat-interval=250 \\
  --election-timeout=2000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

| 字段                              | 含义                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| Working Directory，--data-dir     | 代指工作目录与数据目录为 `${ETCD_DATA_DIR}`，需要启动之前创建目录 |
| --wal-dir                         | 指定 wal 目录，为了提高性能，一般使用 SSD 或者是与 `--data-dir`不同的磁盘 |
| --name                            | 指定节点名称，当 `--initial-cluster-state` 值为 `new` 时，`--name` 的参数值必须位于 `--initial-cluster` 列表中； |
| --cert-file, --key-file           | etcd server  与 client 通信时使用的证书与私钥                |
| --trusted-ca-file                 | 签名 client 证书的 CA 证书，用于验证 client 的证书           |
| --peer-cert-file, --peer-key-file | etcd 与 peer 通信使用的证书和密钥                            |
| --peer-trusted-ca-file            | 签名 peer 证书的 CA 证书，用于验证 peer 证书                 |



### 为各节点创建分发 etcd systemd unit 文件

替换模板文件中的变量, 创建各节点的 systemd unit 文件

- `NODE_NAMES` 和`NODE_IPS` 为相同长度的 bash 数组，分别为节点名称和对应的 IP；

```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 3; i++ ))
  do
    sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" etcd.service.template > etcd-${NODE_IPS[i]}.service 
  done
ls *.service
```

分发生成的 systemd unit 文件

```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    scp etcd-${node_ip}.service root@${node_ip}:/etc/systemd/system/etcd.service
  done
```



### 启动 etcd

- 启动服务之前必须要创建 etcd 数据目录 与 工作目录
- etcd 首次启动会等待其他 etcd 节点加入集群， `systemctl start etcd` 卡住一段时间，属于正常现象

```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "mkdir -p ${ETCD_DATA_DIR} ${ETCD_WAL_DIR}"
    ssh root@${node_ip} "systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd " &
  done
```



### 检查启动并验证服务状态



- 3.4.3 版本的 etcd/etcdctl 默认启用了 V3 API，所以执行 etcdctl 命令时不需要再指定环境变量 `ETCDCTL_API=3`；
- 从 K8S 1.13 开始，不再支持 v2 版本的 etcd；

```shell
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    /opt/k8s/bin/etcdctl \
    --endpoints=https://${node_ip}:2379 \
    --cacert=/etc/kubernetes/cert/ca.pem \
    --cert=/etc/etcd/cert/etcd.pem \
    --key=/etc/etcd/cert/etcd-key.pem endpoint health
  done
```



- 预期输出

```shell
>>> 192.168.7.200
https://192.168.7.200:2379 is healthy: successfully committed proposal: took = 2.439169ms
>>> 192.168.7.201
https://192.168.7.201:2379 is healthy: successfully committed proposal: took = 2.030526ms
>>> 192.168.7.202
https://192.168.7.202:2379 is healthy: successfully committed proposal: took = 2.46203ms
```



### 查看当前 Leader

- 通过 `IS LEADER` 字段可以判断出当前哪个节点是 LEADER 节点

```shell
source /opt/k8s/bin/environment.sh
/opt/k8s/bin/etcdctl \
  -w table --cacert=/etc/kubernetes/cert/ca.pem \
  --cert=/etc/etcd/cert/etcd.pem \
  --key=/etc/etcd/cert/etcd-key.pem \
  --endpoints=${ETCD_ENDPOINTS} endpoint status 
```

```shell
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.7.200:2379 | 4e420e95e8f0acdf |   3.4.3 |   16 kB |      true |      false |         2 |          8 |                  8 |        |
| https://192.168.7.201:2379 | 90c0bee619181d8b |   3.4.3 |   16 kB |     false |      false |         2 |          8 |                  8 |        |
| https://192.168.7.202:2379 | 9a039738e256b955 |   3.4.3 |   16 kB |     false |      false |         2 |          8 |                  8 |        |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

```






















