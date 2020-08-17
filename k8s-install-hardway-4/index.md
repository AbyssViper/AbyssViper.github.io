# K8s Install Hardway 4


## 前言

### 系列目录

- [二进制部署Kubernetes Part1](https://www.viper.pub/k8s-install-hardway-1/)
- [二进制部署Kubernetes Part2](https://www.viper.pub/k8s-install-hardway-2/)
- [二进制部署Kubernetes Part3(本篇)](https://www.viper.pub/k8s-install-hardway-3/)
- [二进制部署Kubernetes Part4](https://www.viper.pub/k8s-install-hardway-4/)

### 注意点

- 本章节所有操作默认均在 `k8s-node01`下操作



## CoreDNS

### 下载

```shell
cd /opt/k8s/work
git clone https://github.com/coredns/deployment.git
mv deployment coredns-deployment
```



### 创建CoreDNS

```shell
cd /opt/k8s/work/coredns-deployment/kubernetes
source /opt/k8s/bin/environment.sh
./deploy.sh -i ${CLUSTER_DNS_SVC_IP} -d ${CLUSTER_DNS_DOMAIN} | kubectl apply -f -
```

### 检查功能

确保 Ready 与 Status正常

```shell
kubectl get all -n kube-system -l k8s-app=kube-dns
```



新建一个Deployment 并发布

```shell
cd /opt/k8s/work
cat > my-nginx.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
EOF
kubectl create -f my-nginx.yaml
```



export 该 Deployment 生成的 ，生成 my-nginx 服务

```shell
kubectl expose deploy my-nginx
```

```shell
kubectl get services my-nginx -o wide


```




























