# CentOS7 NFS 服务安装配置

+ 服务端安装
    ```
    # yum -y install nfs-utils
    # mkdir -p /data/k8s/dynamic
    # chown -R nfsnobody.nfsnobody /data/k8s/dynamic
    ```
+ 服务端配置
    ```
    # vi /etc/exports

    /data/k8s/dynamic *(insecure,rw,async,no_root_squash)
    ```
+ 客户端挂载、卸载
    ```
    # mount -t nfs 10.0.8.2:/data/k8s/dynamic /mnt
    # umount /mnt
    ```