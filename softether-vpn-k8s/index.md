# Softether VPN 打通容器调试网络


## 介绍

Kubernetes 提供了多种对外暴露服务的方式，但是对于调试环境来说，使用 VPN 方式打通与POD之间的隔阂，会变得十分方便





## 部署

使用镜像 `abyssviper/softethervpn` ，该镜像编译基于 `alpine`  基础镜像编译，非常轻量

并且去除了 拆分隧道 限制（参考该文章：），可以向 VPN 客户端推送定制路由



Deployment 镜像编译
