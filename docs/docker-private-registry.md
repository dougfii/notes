# docker 私有仓库
+ 安装
```
# docker pull registry
# docker volume create --name registry
# docker run -p 5000:5000 --name registry --restart=always --privileged=true -v registry:/var/lib/registry -d registry
```
+ 配置
```
# vi /etc/docker/daemon.json

{"insecure-registries":["10.0.8.2:5000"]}
```