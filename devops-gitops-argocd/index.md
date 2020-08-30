# Devops Gitops Argocd












kubectl create ns devops





```shell
$ helm repo add argo https://argoproj.github.io/argo-helm
$ helm install argocd -n argocd argo/argo-cd --values values.yaml
```

![image-20200826093202699](C:\Users\AbyssViper\AppData\Roaming\Typora\typora-user-images\image-20200826093202699.png)

helm ls -n devops



```bash
kubectl get pods -n argocd
```
