# MiniKube 安装
参考地址：<https://yq.aliyun.com/articles/221687>

+ 安装
```
# minikube start --registry-mirror=https://registry.docker-cn.com --vm-driver="hyperv" --hyperv-virtual-switch="ExtSwitch" --cpus=4 --memory=8196 --insecure-registry=10.0.8.1:5000
# minikube dashboard
```
+ 删除
```
# minikube delete
# rm -rf ~/.kube
```
+ 查看IP
```
# minikube ip
```
+ 打开服务
```
# minikube service tomcat
```
+ 一些基本常用命令
```
== create deployment
# kubectl run tomcat --image=tomcat --port=8080 --replicas=3
# kubectl get deployments

== scale deployment
# kubectl scale deployments/tomcat --replicas=4

== crate service (NodePort、LoadBalancer、ClusterIP)
# kubectl expose deployments/tomcat --type=NodePort --port 8080

== view logs & env & into
# kubectl logs tomcat-xxxx
# kubectl exec tomcat-xxxx env
# kubectl run -it tomcat-xxxx bash

== set pod label
# kubectl label pod tomcat-xxxx app=v1

== get & describe
# kubectl get deloyments
# kubectl get services
# kubectl get pods
# kubectl get pods -l run=tomcat
# kubectl get pods -l app=v1
# kubectl get pods -o wide
# kubectl get rc
# kubectl describe pods
# kubectl describe services/tomcat

== delete
# kubectl delete deployment tomcat
# kubectl delete service tomcat
# kubectl delete service -l app=v1
# kubectl delete service -l run=tomcat
# kubectl delete rc tomcat
# kubectl delete replicationcontroller tomcat
# kubectl delete pod tomcat-xxxx
```