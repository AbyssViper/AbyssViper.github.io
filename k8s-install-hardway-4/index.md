# Kubernetes 二进制部署 Part4


## 前言

### 系列目录

- [二进制部署Kubernetes Part1](https://www.viper.pub/k8s-install-hardway-1/)
- [二进制部署Kubernetes Part2](https://www.viper.pub/k8s-install-hardway-2/)
- [二进制部署Kubernetes Part3](https://www.viper.pub/k8s-install-hardway-3/)
- [二进制部署Kubernetes Part4(本篇)](https://www.viper.pub/k8s-install-hardway-4/)

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
$ kubectl get services my-nginx -o wide

NAME       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
my-nginx   ClusterIP   10.254.251.214   <none>        80/TCP    10s   run=my-nginx
```

创建另一个 Pod，查看 `/etc/resolv.conf` 是否包含 `kubelet` 配置的 `--cluster-dns` 和 `--cluster-domain`，是否能够将服务 `my-nginx` 解析到上面显示的 Cluster IP `10.254.251.214`

```shell
cd /opt/k8s/work
cat > dnsutils-ds.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: dnsutils-ds
  labels:
    app: dnsutils-ds
spec:
  type: NodePort
  selector:
    app: dnsutils-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dnsutils-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: dnsutils-ds
  template:
    metadata:
      labels:
        app: dnsutils-ds
    spec:
      containers:
      - name: my-dnsutils
        image: tutum/dnsutils:latest
        command:
          - sleep
          - "3600"
        ports:
        - containerPort: 80
EOF
kubectl create -f dnsutils-ds.yml
```

```shell
$ kubectl get pods -lapp=dnsutils-ds -o wide 

NAME                READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
dnsutils-ds-2dpjw   1/1     Running   0          33m   172.30.135.132   k8s-node03   <none>           <none>
dnsutils-ds-85d6q   1/1     Running   0          33m   172.30.85.195    k8s-node01   <none>           <none>
dnsutils-ds-nwnz4   1/1     Running   0          33m   172.30.58.195    k8s-node02   <none>           <none>
```



查看 `/etc/resolv.conf`

```shell
$ kubectl -it exec dnsutils-ds-2dpjw  cat /etc/resolv.conf

search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.254.0.2
options ndots:5
```
通过 nslookup 进行解析测试

```shell
$ kubectl -it exec dnsutils-ds-2dpjw nslookup kubernetes

Server:		10.254.0.2
Address:	10.254.0.2#53

Name:	kubernetes.default.svc.cluster.local
Address: 10.254.0.1
```

```shell
$ kubectl -it exec dnsutils-ds-2dpjw nslookup www.baidu.com

Server:		10.254.0.2
Address:	10.254.0.2#53

Non-authoritative answer:
www.baidu.com	canonical name = www.a.shifen.com.
Name:	www.a.shifen.com
Address: 180.97.34.94
Name:	www.a.shifen.com
Address: 180.97.34.96
```

```shell
$ kubectl -it exec dnsutils-ds-2dpjw nslookup my-nginx

Server:		10.254.0.2
Address:	10.254.0.2#53

Name:	my-nginx.default.svc.cluster.local
Address: 10.254.251.214
```



## Dashborad 插件

### 下载配置

```shell
cd /opt/k8s/work
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc4/aio/deploy/recommended.yaml
mv  recommended.yaml dashboard-recommended.yaml
```



### 执行所有定义文件

```shell
cd /opt/k8s/work
kubectl apply -f dashboard-recommended.yaml
```



### 检测运行状态

```shell
$ kubectl get pods -n kubernetes-dashboard 

NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-7b8b58dc8b-djtvn   1/1     Running   0          16m
kubernetes-dashboard-6cfc8c4c9-pw2sx         1/1     Running   0          16m
```



### 访问 Dashboard

Kubernetes 1.7 以后，dashboard 只允许通过 https访问。如果使用 kube proxy 则必须监听 localhost 或 127.0.0.1。对于 NodePort 没有这个限制，但是仅建议在开发环境中使用。对于不满足这些条件的登录访问，在登录成功后**浏览器不跳转，始终停在登录界面**

#### 通过 port forward 访问 dashboard

```shell
kubectl port-forward -n kubernetes-dashboard  svc/kubernetes-dashboard 4443:443 --address 0.0.0.0
```

![](https://gitee.com/AbyssViper/pic/raw/master/images/k8s-install-hardway/dashboard-login-view.png)



### 创建登录 Dashboard 的 token 和 kubeconfig 文件

dashboard 默认只 支持 token 认证（不支持 client 证书认证），所以如果使用 Kubeconfig 文件，需要将 token 写入到该文件



#### 创建登录 token

使用输出的 token 登录

```shell
kubectl create sa dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep dashboard-admin | awk '{print $1}')
DASHBOARD_LOGIN_TOKEN=$(kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}')
echo ${DASHBOARD_LOGIN_TOKEN}
```



#### 创建使用 token 的 Kubeconfig 文件

用生成的 dashboard.kubeconfig 登录 Dashboard

```shell
source /opt/k8s/bin/environment.sh
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/cert/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=dashboard.kubeconfig

# 设置客户端认证参数，使用上面创建的 Token
kubectl config set-credentials dashboard_user \
  --token=${DASHBOARD_LOGIN_TOKEN} \
  --kubeconfig=dashboard.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=dashboard_user \
  --kubeconfig=dashboard.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=dashboard.kubeconfig
```

![](https://gitee.com/AbyssViper/pic/raw/master/images/k8s-install-hardway/dashboard-view.png)



## 部署 kube-prometheus 框架

kube-prometheus 是一整套监控解决方案，它使用 Prometheus 采集集群指标，Grafana 做展示，包含如下组件：

- The Prometheus Operator
- Highly available Prometheus
- Highly available Alertmanager
- Prometheus node-exporter
- Prometheus Adapter for Kubernetes Metrics APIs （k8s-prometheus-adapter）
- kube-state-metrics
- Grafana

其中 k8s-prometheus-adapter 使用 Prometheus 实现了 metrics.k8s.io 和 custom.metrics.k8s.io API，所以**不需要再部署** `metrics-server`。

### 下载和安装

```shell
cd /opt/k8s/work
git clone https://github.com/coreos/kube-prometheus.git
cd kube-prometheus/
sed -i -e 's_quay.io_quay.mirrors.ustc.edu.cn_' manifests/*.yaml manifests/setup/*.yaml # 使用中科大的 Registry
kubectl apply -f manifests/setup # 安装 prometheus-operator
kubectl apply -f manifests/ # 安装 promethes metric adapter
```



### 查看运行结果

```shell
$ kubectl get pods -n monitoring

NAME                                   READY   STATUS    RESTARTS   AGE
alertmanager-main-0                    2/2     Running   0          14h
alertmanager-main-1                    2/2     Running   0          14h
alertmanager-main-2                    2/2     Running   0          14h
grafana-85c89999cb-cxzsd               1/1     Running   0          14h
kube-state-metrics-79f47cd6dc-8h92p    3/3     Running   0          14h
node-exporter-b584n                    2/2     Running   0          14h
node-exporter-jmcbk                    2/2     Running   0          14h
node-exporter-zwc8h                    2/2     Running   0          14h
prometheus-adapter-b8d458474-r4brv     1/1     Running   0          14h
prometheus-k8s-0                       3/3     Running   0          14h
prometheus-k8s-1                       3/3     Running   0          14h
prometheus-operator-784c457694-k9s7q   2/2     Running   0          14h
```



```shell
$ kubectl top pods -n monitoring

NAME                                   CPU(cores)   MEMORY(bytes)   
alertmanager-main-0                    2m           43Mi            
alertmanager-main-1                    2m           41Mi            
alertmanager-main-2                    1m           45Mi            
grafana-85c89999cb-cxzsd               8m           44Mi            
kube-state-metrics-79f47cd6dc-8h92p    0m           65Mi            
node-exporter-b584n                    2m           34Mi            
node-exporter-jmcbk                    4m           37Mi            
node-exporter-zwc8h                    3m           38Mi            
prometheus-adapter-b8d458474-r4brv     2m           38Mi            
prometheus-k8s-0                       68m          489Mi           
prometheus-k8s-1                       51m          490Mi           
prometheus-operator-784c457694-k9s7q   0m           51Mi  
```



### 访问测试

启动代理，访问 Prometheus

- port-forward 依赖 socat
- 访问 http://192.168.7.200:9090/new/graph?g0.expr=&g0.tab=1&g0.stacked=0&g0.range_input=1h

```shell
kubectl port-forward --address 0.0.0.0 pod/prometheus-k8s-0 -n monitoring 9090:9090
```

![](https://gitee.com/AbyssViper/pic/raw/master/images/k8s-install-hardway/prometheus-preview.png)



启动代理，访问 Grafana

- 访问 http://192.168.7.200:3000/
- 默认账号密码   admin/admin，登录后需要修改密码

```shell
kubectl port-forward --address 0.0.0.0 svc/grafana -n monitoring 3000:3000 
```

![](https://gitee.com/AbyssViper/pic/raw/master/images/k8s-install-hardway/grafana-review.png)



## 部署 EFK 插件

### 修改配置文件

EFK 目录是 `kubernetes/cluster/addons/fluentd-elasticsearch`

```shell
cd /opt/k8s/work/kubernetes/cluster/addons/fluentd-elasticsearch
sed -i -e 's_quay.io_quay.mirrors.ustc.edu.cn_' es-statefulset.yaml # 使用中科大的 Registry
sed -i -e 's_quay.io_quay.mirrors.ustc.edu.cn_' fluentd-es-ds.yaml # 使用中科大的 Registry
```



### 执行定义文件

```shell
cd /opt/k8s/work/kubernetes/cluster/addons/fluentd-elasticsearch
kubectl apply -f .
```



### 结果检查

```shell
$ kubectl get all -n kube-system |grep -E 'elasticsearch|fluentd|kibana'

pod/elasticsearch-logging-0                    1/1     Running   0          57m
pod/elasticsearch-logging-1                    1/1     Running   0          27m
pod/fluentd-es-v2.7.0-8d2d8                    1/1     Running   0          57m
pod/fluentd-es-v2.7.0-bt9rk                    1/1     Running   0          57m
pod/fluentd-es-v2.7.0-r2h79                    1/1     Running   0          57m
pod/kibana-logging-75888755d6-vrdsc            0/1     Running   0          57m
service/elasticsearch-logging   ClusterIP   10.254.198.164   <none>        9200/TCP                       57m
service/kibana-logging          ClusterIP   10.254.201.178   <none>        5601/TCP                       57m
daemonset.apps/fluentd-es-v2.7.0   3         3         3       3            3           <none>                   57m
deployment.apps/kibana-logging            0/1     1            0           57m
replicaset.apps/kibana-logging-75888755d6            1         1         0       57m
statefulset.apps/elasticsearch-logging   2/2     57m
```

kibana Pod 第一次启动时会用**较长时间(0-20分钟)**来优化和 Cache 状态页面，可以 tailf 该 Pod 的日志观察进度：

```
kubectl logs kibana-logging-75888755d6-vrdsc -n kube-system -f
```

注意：只有当 Kibana pod 启动完成后，浏览器才能查看 kibana dashboard，否则会被拒绝。



### 访问测试

- 访问 http://192.168.7.200:8086/api/v1/namespaces/kube-system/services/kibana-logging/proxy

```shell
kubectl proxy --address='192.168.7.200' --port=8086 --accept-hosts='^*$'
```

在 Management -> Indices 页面创建一个 index（相当于 mysql 中的一个 database），选中 `Index contains time-based events`，使用默认的 `logstash-*` pattern，点击 `Create` ;

![](https://gitee.com/AbyssViper/pic/raw/master/images/k8s-install-hardway/kibana-patterns.png)



创建 Index 后，稍等几分钟就可以在 `Discover` 菜单下看到 ElasticSearch logging 中汇聚的日志；

![](https://gitee.com/AbyssViper/pic/raw/master/images/k8s-install-hardway/kibana-discover.png)




















