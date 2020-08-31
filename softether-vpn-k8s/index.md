# Softether VPN 打通容器调试网络


## 介绍

Kubernetes 提供了多种对外暴露服务的方式 

对于开发调试来说，使用 VPN 方式打通 **本机开发环境 与 Cluster** 之间的网络，会变得十分方便

### SoftEther VPN

选用 `SoftEther VPN` 主要原因是

- 支持通过**单端口多种VPN接入方式**，比如 `SoftEtherVPN Client`、`OpenVPN`
- 有统一的账户管理、分组管理、权限控制、拆分隧道 等功能

### 部署介绍

- 自定义部署：支持拆分隧道功能，需要下载客户端进行管理
- 无配置快速部署：不支持拆分隧道功能，可以不连接远程管理，开箱即用



## 自定义部署

使用镜像 `abyssviper/softethervpn` ，该镜像编译基于 `alpine`  基础镜像编译，仅保留了 `vpnserver`, 非常轻量

并且去除了 拆分隧道 限制（[参考该文章](https://www.viper.pub/softether-vpn-split-tunnel/)），可以向 VPN 客户端**推送定制路由**



### 完整配置文件

创建发布配置文件  deployment-softethervpn.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpn
  namespace: devops
spec:
  selector:
   matchLabels:
     app: softether-vpnserver
  template:
    metadata:
      labels:
        app: softether-vpnserver
    spec:
      containers:
      - name: softether-vpn-alpine
        image: abyssviper/softethervpn
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5555
          name: connect
          protocol: TCP
        livenessProbe:
          tcpSocket:
            port: 5555
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        readinessProbe:
          tcpSocket:
            port: 5555
        resources:
          limits:
            cpu: 1000m
            memory: 200Mi
          requests:
            cpu: 500m
            memory: 100Mi
        volumeMounts:
        - name: softether-vpn-storge
          subPath: softethervpn/vpn_server.config
          mountPath: /opt/vpnserver/vpn_server.config
        - name: softether-vpn-storge
          subPath: softethervpn/server_log
          mountPath: /opt/vpnserver/server_log
        - name: softether-vpn-storge
          subPath: softethervpn/packet_log
          mountPath: /opt/vpnserver/packet_log
        - name: softether-vpn-storge
          subPath: softethervpn/security_log
          mountPath: /opt/vpnserver/security_log
      volumes:
      - name: softether-vpn-storge
        persistentVolumeClaim:
          claimName: vpn-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: vpn
  namespace: devops
spec:
  selector:
    app: softether-vpnserver
  type: NodePort
  ports:
  - name: connect
    port: 5555
    nodePort: 30003
```

创建PV, PVC配置文件  pvc-softethervpn.yaml  本文使用 NFS

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vpn-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  nfs:
    server: 192.168.7.40
    path: /data/kubernetes

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vpn-pvc
  namespace: devops
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```



首先创建发布 PV, PVC 配置

```shell
$ kubectl apply -f pvc-softethervpn.yaml
```

然后发布SoftEtherVPN

```shell
$ kubectl apply -f deployment-softethervpn.yaml
```

### 管理

#### 管理工具下载

通过 `SoftEtherVPN` 官方提供的 Manager 工具进行管理（[下载地址](https://www.softether-download.com/cn.aspx?product=softether)）

![管理工具下载地址](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/softethervpn-manager-download.png "管理工具下载地址")

<br />

#### 连接配置

配置服务端**地址以及端口**， 第一次连接需要手动设置管理密码

![服务端连接配置](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/softethervpn-manager-connect-config.png "服务端连接配置")

<br />

#### 配置 SecureNAT 和 用户

连接成功后，通过 **管理虚拟HUB(A)** —》**虚拟 NAT 和 虚拟 DHCP 服务器(V)** —》**启用 SecureNAT(E)** —》**SecureNAT配置(C)**

![SecureNAT配置](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/securenat-config.png "SecureNAT配置")

- **网关配置**：如果填写，则本地VPN客户端流量默认路由走服务端，不需要推送路由；推荐使用本地网关+路由推送的方式
- **DNS配置**：如果需要使用DNS解析（example: my-nginx.default.svc.cluster.local），需要配置集群内 `CoreDNS ` 的 IP 地址
- **路由推送**：路由推送格式如图所示，网关地址填写 SecureNAT 配置的网关地址

<br />

在  **管理虚拟HUB(A)** 中，添加一个用户

![添加用户](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/hub-config-add-user.png "添加用户")



#### 配置OpenVPN

通过 VPN 管理工具，**启用 OpenVPN 功能**，生成 OpenVPN Client 配置样本文件

![OpenVPN配置](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/openvpn-config.png "OpenVPN配置")



解压后，对 `access_l3.ovpn` 进行编辑



![OpnVPN配置案例文件](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/openvpn-config-template.png "OpnVPN配置案例文件")

按照如下的配置进行修改，其中

- **remote**: 上述 Kubernetes 发布 VPN 配置文件的 `Service` 地址
- **proto**: 使用 TCP 作为连接协议，也可以使用 UDP，注意修改发布 VPN 时 YAML 配置文件的 端口映射配置
- **ca**:  添加 `ca`标签，填写 VPN Server 配置中的 证书信息，证书信息获取方式见后面

```shell
dev tun
proto tcp
remote 192.168.7.200 30003
cipher AES-128-CBC
auth SHA1
resolv-retry infinite
nobind
persist-key
persist-tun
client
verb 3
auth-user-pass
<ca>
-----BEGIN CERTIFICATE-----
从如下配置中获取
-----END CERTIFICATE-----
</ca>
```

可以通过打开 **编辑设置(D)** 找到证书信息 或  **加密与网络(E)** 导出配置文件

![获取ca信息方式1](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/openvpn-config-cert1.png "获取ca信息方式1")

![获取ca信息方式2](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/openvpn-config-cert2.png "获取ca信息方式2")



### 连接

#### SoftEtherVPN Client

通过官网下载连接客户端，适用于 Windows 方式，Linux中也可以；推荐 Linux 与 OSX 中使用 OpenVPN 的方式

#### OpenVPN





### 持久化



## 无配置快速部署










