# Kubernetes DevOps Jenkins 高可用部署


https://www.cnblogs.com/zhongyuanzhao000/p/11430183.html

https://www.qikqiak.com/post/kubernetes-jenkins1/

## 结构图

![](https://gitee.com/AbyssViper/pic/raw/master/images/devops-k8s-jenkins-ha/jenkins-ha.png)





## 部署

部署步骤

- 创建 namespaces
- 创建 pv，pvc 
- 创建 rbac account
- 生成 deployment 以及 service









```shell
kubectl create namespace devops
```



跳过

```shell
/restart
```





## 配置

### 初始化






















