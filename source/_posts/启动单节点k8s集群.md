---
title: 启动单节点k8s集群
date: 2018-08-17 13:38:46
tags: k8s
categories: k8s
---

# 安装minikube

因为国内网络的原因通过阿里云的源下载

```
curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v0.28.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

```
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && chmod +x kubectl && sudo cp kubectl /usr/local/bin/ && rm kubectl
```

这个时候已经下载好了minikube，尝试使用命令，并通过运行minikube start命令启动集群：

```
minikube version
minikube start --bootstrapper=localkube
```
**非虚拟机启动 minikube start --vm-driver=none**


可以使用kubectl CLI与集群进行交互。这是用于管理Kubernetes和在集群顶部运行的应用程序的主要方法。 可以通过以下方式发现群集及其运行状况的详细信息

```
kubectl get nodes
```

> $ kubectl get nodes
> NAME       STATUS     ROLES     AGE       VERSION
> minikube   NotReady   <none>    8s        v1.10.0

如果节点标记为NotReady，则它仍在启动组件。 此命令显示可用于托管应用程序的所有节点。现在我们只有一个节点，我们可以看到它的状态已准备就绪（它已准备好接受部署的应用程序）。

通过运行Kubernetes集群，现在可以部署容器。 使用`kubectl run`，它允许将容器部署到集群上 

```shell
kubectl run first-deployment --image=katacoda/docker-http-server --port=80
```

可以通过正在运行的Pod发现部署状态 - 

> $ kubectl get pods
> NAME                                                     READY     STATUS    RESTARTS   AGE
> first-deployment-59f6bb4956-qvdg2   1/1       Running           0          1m

容器运行后，可以根据需要通过不同的网络选项进行暴露。一种可能的解决方案是NodePort，它为容器提供动态端口。

```shell
kubectl expose deployment first-deployment --port=80 --type=NodePort
```

以下命令查找已分配的端口并执行HTTP请求

```shell
export PORT=$(kubectl get svc first-deployment -o go-template='{{range.spec.ports}}{{if .nodePort}}{{.nodePort}}{{"\n"}}{{end}}{{end}}')
echo "Accessing host01:$PORT"
curl host01:$PORT
```

























