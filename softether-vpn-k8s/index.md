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

并且去除了 拆分隧道 限制（[参考该文章](https://www.viper.pub/softether-vpn-split-tunnel/)），可以向 VPN 客户端  **推送定制路由**



### 完整配置文件

创建发布配置文件  deployment-softethervpn.yaml, 配置文件的 Deployment 命名存在一定问题，请看**本章疑问点部分**

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
- **路由推送**：路由推送格式如图所示，`IP网络地址/子网掩码/网关IP地址`，网关IP地址填写 SecureNAT 配置的网关地址; 一般需要推送 Kubernetes **Calico 网段 以及 SVC网段**

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

通过官网[下载连接客户端](https://www.softether-download.com/?product=softether)，适用于 `Windows / Linux`；推荐 Linux 与 OSX 中使用 `OpenVPN` 的方式

- 需要填写 **主机名、端口号、虚拟HUB名、账号密码**
- `SoftEtherVPN Client` 支持同时连接多个VPNServer，只需要创建多个`虚拟网络适配器`即可

![SoftEtherVPN Client 连接配置](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/softethervpn-client-config.png "SoftEtherVPN Client 连接配置")

#### OpenVPN

下载Open VPN 客户端： [Windows下载地址](http://www.canadiancontent.net/tech/download/OpenVPN_GUI.html)， [OSX下载地址](https://tunnelblick.en.softonic.com/mac)

通过上述配置的配置文件即可进行连接

- SoftEtherVPN Server 可以设置**多个虚拟 HUB**；对于OpenVPN来说，通过 **用户名@HUB** 的方式指定 HUB，直接使用用户名默认是 `Default` HUB
- 例如：VPN HUB中的 test 用户，用户名为：`test@VPN`

![OpenVPN 客户端连接](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/openvpn-client-config.png "OpenVPN 客户端连接")

### 验证

VPN连接成功后，可以进行 **DNS 解析设置** 以及 **访问测试**

![DNS解析设置](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/test-dns.png "DNS解析设置")

以 `ArgoCD` 为例进行**访问测试**

![访问测试](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/test-argocd.png "访问测试")



### 数据持久化

- **配置的持久化**: 只需要持久化 `vpn_server.config` 即可
- **日志的持久化**: `server_log` `packet_log` `security_log` 三个文件夹持久化即可
- **注意点**: 需要注意的是，SoftEtherVPN Server **并不会在更改配置（添加用户，更改SecuretNAT等）后马上**写入 `vpn_server.config`, 需要等待一段时间间隔才会落地到 `vpn_server.config` 文件



## 无配置快速部署

快速部署配置，选型的镜像为 `siomiz/softethervpn`, 该镜像同样有基于 `alpine` 版本，并且会**启动容器时进行快速初始化**，并创建可用的账户，非常方便；不过该镜像 不支持拆分隧道功能 ，无法推送定制路由



### 配置文件

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
        image: siomiz/softethervpn
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

发布到 Kubernetes 中
```shell
$ kubectl apply -f deployment-softethervpn.yaml
```

### 查看连接信息

通过 `logs` 查看日志信息，从中查看默认生成的 SoftEtherVPN 账户 以及 OpenVPN 配置文件

```shell
$ kubectl logs -f -n devops vpn-b55fb8f4-jmxqh
```
如下，可以看出 用户名为 `user9703` 密码为 `6758.1071.6532.2086.9735`, OpenVPN 配置文件 等信息

- OpenVPN 连接的 `proto` 以及 `remote` 信息需要更改为合适的，具体参照自定义部署

```shell
# [!!] This image requires --cap-add NET_ADMIN
# ========================
# user9703
# 6758.1071.6532.2086.9735
# ========================
# Version 4.34 Build 9745   (English)
dev tun
proto udp
remote _unregistered_vpn528125132.v4.softether.net 1194
;http-proxy-retry
;http-proxy [proxy server] [proxy port]
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
MIIDyjCCArKgAwIBAgIBADANBgkqhkiG9w0BAQsFADBkMRswGQYDVQQDDBJ2cG4t
YjU1ZmI4ZjQtam14cWgxGzAZBgNVBAoMEnZwbi1iNTVmYjhmNC1qbXhxaDEbMBkG
A1UECwwSdnBuLWI1NWZiOGY0LWpteHFoMQswCQYDVQQGEwJVUzAeFw0yMDA4MzEx
NjM2NDNaFw0zNzEyMzExNjM2NDNaMGQxGzAZBgNVBAMMEnZwbi1iNTVmYjhmNC1q
bXhxaDEbMBkGA1UECgwSdnBuLWI1NWZiOGY0LWpteHFoMRswGQYDVQQLDBJ2cG4t
YjU1ZmI4ZjQtam14cWgxCzAJBgNVBAYTAlVTMIIBIjANBgkqhkiG9w0BAQEFAAOC
AQ8AMIIBCgKCAQEA9j++0cYr7/1enukSjhzA37s01SWNazUcpgEjrclfikuzKiw0
M7bJGEjM8eJTUqvtIwJOkWVbrVfVTX1zV/yCenFns05WRSud2oEGyXWh0oa8aChv
w/S+KYdGub4sLkwDbIfGEhJQIXO3iQ9ecdjX+QFUlOL7PdCDyxc6wao2ZsjwCeLt
oamj8AOVH+w0E24OC0H3eiJ5YMKWo56JwH0spbwl/xONq1PfUuP494dG6C7sOMWS
DIW3OD3Bo071B9A5OGtE/fRUe56ZxsOZySlhaI1Yl8LZvZtSdkAhLByKYTjmKd7J
NbCWJUMiLCSIxRFAjxCDjmBrEBGkAtM4v+PC1wIDAQABo4GGMIGDMA8GA1UdEwEB
/wQFMAMBAf8wCwYDVR0PBAQDAgH2MGMGA1UdJQRcMFoGCCsGAQUFBwMBBggrBgEF
BQcDAgYIKwYBBQUHAwMGCCsGAQUFBwMEBggrBgEFBQcDBQYIKwYBBQUHAwYGCCsG
AQUFBwMHBggrBgEFBQcDCAYIKwYBBQUHAwkwDQYJKoZIhvcNAQELBQADggEBALRC
1HKokh3KwpgKjznMwOR83bPu8QveHWr0GrlzseKxqHGJcTy0sxnkfk3mAu9v8m4a
UACj3H0opouRAqOTdbogCWXcERwLM1084wehyeUZKX9gfcWGbAPWVjcY1kC5KePs
IXWhEMC56wIGMFs4mS5vx7aNVE9k4Ssrnf7T3mkM/ACrN9dg+/H2CVxNr5FTQIwy
IGTC3AP5WLPVfEk5SByEOZqFRiBIDDhvKU4gT4cD2+FHLM6OM8Z09qGs8uq6KLr6
LfUjc/c5CI+FInmm1hLB3NZug17TEaVchXeQNs921wKOOWoCKucToOPXkwYE1V/c
zc7doXSaSkrKyIwCqCY=
-----END CERTIFICATE-----
</ca>
;<cert>
;-----BEGIN CERTIFICATE-----
;
;-----END CERTIFICATE-----
;</cert>
;<key>
;-----BEGIN RSA PRIVATE KEY-----
;
;-----END RSA PRIVATE KEY-----
;</key>
# Creating user(s): user9703
# [initial setup OK]
```

### 验证

SoftEtherVPN Client 连接后，访问验证

![SoftEtherVPN Client验证](https://gitee.com/AbyssViper/pic/raw/master/images/softether-vpn-k8s/test-quick-install.png "SoftEtherVPN Client验证")

### 其他配置

该镜像提供了很多初始化配置信息，具体可以[参考官方配置信息](https://hub.docker.com/r/siomiz/softethervpn)



## 疑问点

上述的 Deployment 配置文件存在如下问题：

- 部分 `deployment name` 会导致 **开启 SecureNAT** 以后， 客户端连接后，立即断开 客户端与服务端的 连接
- name 为 `vpn` `centos7 `  `vpn-s` 之类测试的可以正常使用，`vpnserver ` `vpn-server` `softether-vpn` 之类的就会断开
- 该问题暂时没有找到原因，如果有了解的，期望您留言指教，感谢






