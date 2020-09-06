# Operator Sdk Shallow








环境 linux  。下载 linux  version

添加执行权限， 移动到  /usr/bin



operator-sdk version



安装本地 harbor 或者  docker registry

docker run -d -p 5000:5000 --restart always  -v /data/registry:/var/lib/registry  --name registry registry:latest

```shell
docker run -d -p 5000:5000 --restart always --name registry -v /data/registry:/var/lib/registry registry:latest
```



```shell
cat /etc/docker/daemon.json 
{
    "insecure-registries": ["localhost:5000"],
    "registry-mirrors": ["https://tqsl0ohv.mirror.aliyuncs.com"]
}
```

查看镜像库中的镜像列表


```shell
http://192.168.36.138:5000/v2/_catalog
```







```shell
git clone https://gitee.com/mirrors/Kubernetes.git
cp -R Kubernetes/staging/src/k8s.io $GOPATH/src/k8s.io

cd $GOPATH/src
mkdir sigs.k8s.io && cd sigs.k8s.io

git clone https://github.com/kubernetes-sigs/controller-runtime
mkdir yaml
```



快速创建  Operator

```shell
operator-sdk new imoocpod-operator --skip-validation=true --repo=github.com/imooc-com/imoocpod-operator
cd imoocpod-operator
operator-sdk add api --api-version=k8s.imooc.com/v1alpha1 --kind=ImoocPod
operator-sdk add controller --api-version=k8s.imooc.com/v1alpha1 --kind=ImoocPod
```

![image-20200904160644087](C:\Users\AbyssViper\AppData\Roaming\Typora\typora-user-images\image-20200904160644087.png)





主要开发任务在 pkg 中



编译 push 镜像

```shell
operator-sdk build localhost:5000/imoocpod-operator
docker push localhost:5000/imoocpod-operator
```





```shell
kubectl apply -f deploy/service_account.yaml
kubectl apply -f deploy/role.yaml
kubectl apply -f deploy/role_binding.yaml

kubectl apply -f deploy/crds/k8s.imooc.com_imoocpods_crd.yaml


需要修改images name
kubectl apply -f deploy/operator.yaml

kubectl get pod 

kubectl apply -f deploy/crds/k8s.imooc.com_v1alpha1_imoocpod_cr.yaml

```





```shell
api/x/version/*_types.go
```

![image-20200904211102836](C:\Users\AbyssViper\AppData\Roaming\Typora\typora-user-images\image-20200904211102836.png)

期望值添加副本字段，通过该接口体可以映射 Pod 配置文件中的  spec 中的字段



![image-20200904211305803](C:\Users\AbyssViper\AppData\Roaming\Typora\typora-user-images\image-20200904211305803.png)



添加Replicas 以及 PodNames   Pod的期望状态，副本与Pod名称







```shell
operator-sdk generate k8s
operator-sdk generate crds


```

更新完成后 crds/k8s.imooc.com_imoocpods_crd.yaml 中的已经添加了刚才结构体中添加的内容



在 Controller中 Reconcile 进行功能开发



```shell
operator-sdk generate k8s
operator-sdk build registry.cn-hangzhou.aliyuncs.com/viper/imoocpod-operater

docker push registry.cn-hangzhou.aliyuncs.com/viper/imoocpod-operater

kubectl apply -f deploy/crds/k8s.imooc.com_imoocpods_crd.yaml	

kubectl apply -f deploy/crds/k8s.imooc.com_v1alpha1_imoocpod_cr.yaml
```



{
    // "insecure-registries": ["registry.cn-hangzhou.aliyuncs.com/viper/operator"],
    "registry-mirrors": ["https://tqsl0ohv.mirror.aliyuncs.com"]
}
















