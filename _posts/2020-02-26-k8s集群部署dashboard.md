---
layout: post
title: Kubernetes集群部署dashboard
categories: [云原生]
description: Kubernetes集群部署dashboard
keywords: Kubernetes, 云原生, dashboard
---

kubernetes dashboard是一个管理k8s集群的ui页面


#### 集群部署
> Mac本机安装docker自带  
> Linux单集群环境使用MiniKube

#### 部署Dashboard(本次环境采用三台机器集群)
>* github项目地址：
https://github.com/kubernetes/dashboard  
>* 官方参考文档：
https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/#deploying-the-dashboard-ui  
##### 确认匹配版本
> 查看k8s集群版本

```
kubectl version
```

> github上查找对应的release版本

```
https://github.com/kubernetes/dashboard
```

##### 下载yaml文件

```
wget http://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc4/aio/deploy/recommended.yaml
```

##### 根据需要更改yaml文件

##### 访问dashboard的方式
>* Nodeport方式访问dashboard，service类型改为NodePort(single 本地)  
>* loadbalacer方式，service类型改为loadbalacer  
>* Ingress方式访问dashboard  
>* API server方式访问 dashboard  （推荐）
>* kubectl proxy方式访问dashboard(本地)

##### 详解API server方式
> 如果Kubernetes API服务器是公开的，并可以从外部访问，那我们可以直接使用API Server的方式来访问，也是比较推荐的方式

```
https://<master-ip>:<apiserver-port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```
> 浏览器返回结果可能为

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "services \"https:kubernetes-dashboard:\" is forbidden: User \"system:anonymous\" cannot get resource \"services/proxy\" in API group \"\" in the namespace \"kube-system\"",
  "reason": "Forbidden",
  "details": {
    "name": "https:kubernetes-dashboard:",
    "kind": "services"
  },
  "code": 403
}
```
> 生成证书

```
# 生成client-certificate-data
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
# 生成client-key-data
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
# 生成p12
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```

> 证书导入浏览器

##### 创建登录用户

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

##### 获取token
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

##### 访问dashboard
> https://test-scrm.medsci.cn:6443/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/


