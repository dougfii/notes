# Kubernets 1.14.0 集群安装（kubeadm）

##### 环境准备
| IP | Role | Machine |
| --- | --- | --- |
| 10.0.6.1 | master | centos7 |
| 10.0.6.11  | node1| centos7 |
| 10.0.6.12  | node2 | centos7 |

1. 所有机器基础配置
    - 修改主机名称
        ```
        # hostnamectl set-hostname xxx
        # sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
        # systemctl stop firewalld && systemctl disable firewalld
        # systemctl stop iptables && systemctl disable iptables
        # systemctl stop postfix && systemctl disable postfix
        # yum install -y ipvsadm
        ```
    - 修改hosts
        ```
        # vi /etc/hosts
        
        10.0.8.10 node1
        10.0.8.11 node2
        10.0.8.12 node3
        ```
    - 关闭swap分区
        ```
        # swapoff -a
        # sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab  或   vi /etc/fstab （注释掉 #/dev/mapper/centos-swap 这行）
        # sysctl --system
        ```
    - 配置转发参数
        ```
        # vi /etc/sysctl.d/k8s.conf
        net.ipv4.ip_forward = 1
        net.ipv6.conf.all.disable_ipv6=1
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
        vm.swappiness=0

        # sysctl --system
        ```
    - 设置 rsyslogd 和 systemd journald
        ```
        # mkdir /var/log/journal
        # mkdir /etc/systemd/journald.conf.d
        # vi /etc/systemd/journald.conf.d/99-pprophet.conf
      
        [Journal]
        Stroage=persistent
        Compress=yes
        SyncIntervalSec=5m
        RateLimitInterval=30s
        RateLimitBurst=1000
        SystemMaxUse=10G
        SystemMaxFileSize=200M
        MaxRetentionSec=2week
        ForwardToSyslog=no
      
        # systemctl restart systemd-journald
        ```
    - 升级内核（CentOS 7.x 自带 3.10.x 内核存在一些 Bug 会导致运行的 docker、k8s 不稳定）
        ```
        # yum install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
        # yum --enablerepo=elrepo-kernel install -y kernel-lt
        # grub2-set-default "CentOS Linux (4.4.200-1.el7.elrepo.x86_64) 7 (Core)"
        ```
    - 增加 k8s.repo
        ```
        # vi /etc/yum.repos.d/k8s.repo
        [kubernetes]
        name=Kubernetes
        baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
        enabled=1
        gpgcheck=1
        repo_gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
        ```
    - 增加 docker-ce.repo
        ```
        # vi /etc/yum.repo.d/docker-ce.repo
        [docker-ce-edge]
        name=Docker CE Edge - $basearch
        baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/edge
        enabled=0
        gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
         
        [docker-ce-edge-debuginfo]
        name=Docker CE Edge - Debuginfo $basearch
        baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/edge
        enabled=0
        gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
         
        [docker-ce-edge-source]
        name=Docker CE Edge - Sources
        baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/edge
        enabled=0
        gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
         
        [docker-ce-test]
        name=Docker CE Test - $basearch
        baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/test
        enabled=0
        gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
         
        [docker-ce-test-debuginfo]
        name=Docker CE Test - Debuginfo $basearch
        baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/test
        enabled=0
        gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
         
        [docker-ce-test-source]
        name=Docker CE Test - Sources
        baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/test
        enabled=0
        gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
         
        [docker-ce-nightly]
        name=Docker CE Nightly - $basearch
        baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/nightly
        enabled=0
        gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
         
        [docker-ce-nightly-debuginfo]
        name=Docker CE Nightly - Debuginfo $basearch
        baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/nightly
        enabled=0
        gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
         
        [docker-ce-nightly-source]
        name=Docker CE Nightly - Sources
        baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/nightly
        enabled=0
        gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
        ```
    - kube-proxy 开启 ipvs 的前置条件
      ```
      # modprobe br_netfilter
      # vi /etc/sysconfig/modules/ipvs.modules
      
      #!/bin/bash
      modprobe -- ip_vs
      modprobe -- ip_vs_rr
      modprobe -- ip_vs_wrr
      modprobe -- ip_vs_sh
      modprobe -- nf_conntrack_ipv4

      # chmod 755 /etc/sysconfig/modules/ipvs.modules && bash
      # /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
      ```
    - 安装组件
        ```
        # yum -y install docker kubelet-1.14.0 kubeadm-1.14.0 kubectl-1.14.0
        # systemctl enable docker
        # systemctl enable kubelet
        # systemctl start docker
        # systemctl start kubelet
      
        ???
        # vi /etc/docker/daemon.json
      
        {
            "exec-opts": ["native.cgroupdriver=systemd"],
            "log-driver": "json-file",
            "log-opts": {
                "max-size": "100m"
            }
        }

        # mkdir -p /etc/systemd/system/docker.service.d
        # systemctl daemon-reload && systemctl restart docker
        ```
    - 下载镜像
        ```
        # vi > pull.sh
      
        #!/bin/bash
        images=(kube-proxy:v1.16.2 kube-scheduler:v1.16.2 kube-controller-manager:v1.16.2 kube-apiserver:v1.16.2 etcd:3.3.15-0 coredns:1.6.2 pause:3.1)
        for imageName in ${images[@]} ; do
        docker pull registry.aliyuncs.com/google_containers/$imageName
        docker tag registry.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
        docker rmi registry.aliyuncs.com/google_containers/$imageName
        done
        
        # chmod +x pull.sh
        # ./pull.sh
        ```
        
2. Master节点配置
    - 初始化master节点
        ```
        (旧版)# kubeadm init --kubernetes-version=v1.16.2 --pod-network-cidr=172.16.0.0/16 --service-cidr=172.17.0.0/24
      
        # kubeadm config print init-defaults > kubeadm-config.yml
        # vi kubeadm-config.yml
      
        apiVersion: kubeadm.k8s.io/v1beta2
        bootstrapTokens:
        - groups:
          - system:bootstrappers:kubeadm:default-node-token
          token: abcdef.0123456789abcdef
          ttl: 24h0m0s
          usages:
          - signing
          - authentication
        kind: InitConfiguration
        localAPIEndpoint:
          advertiseAddress: 10.0.8.11
          bindPort: 6443
        nodeRegistration:
          criSocket: /var/run/dockershim.sock
          name: node1
          taints:
          - effect: NoSchedule
            key: node-role.kubernetes.io/master
        ---
        apiServer:
          timeoutForControlPlane: 4m0s
        apiVersion: kubeadm.k8s.io/v1beta2
        certificatesDir: /etc/kubernetes/pki
        clusterName: kubernetes
        controllerManager: {}
        dns:
          type: CoreDNS
        etcd:
          local:
            dataDir: /var/lib/etcd
        imageRepository: k8s.gcr.io
        kind: ClusterConfiguration
        kubernetesVersion: v1.16.2
        networking:
          dnsDomain: cluster.local
          podSubnet: 172.16.0.0/16
          serviceSubnet: 172.17.0.0/24
        scheduler: {}
        ---
        apiVersion: kubeproxy.config.k8s.io/v1alpha1
        kind: KubeProxyConfiguration
        featureGates:
          SupportIPVSProxyMode: true
        mode: ipvs

        # kubeadm init --config=kubeadm-config.yml
        ```
        成功后保存如下输出（添加work节点用）
        ```
        Your Kubernetes control-plane has initialized successfully!
        
        To start using your cluster, you need to run the following as a regular user:
        
          mkdir -p $HOME/.kube
          sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
          sudo chown $(id -u):$(id -g) $HOME/.kube/config
        
        You should now deploy a pod network to the cluster.
        Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
          https://kubernetes.io/docs/concepts/cluster-administration/addons/
        
        Then you can join any number of worker nodes by running the following on each as root:
        
        kubeadm join 10.0.6.1:6443 --token fmot1q.7oicgu8h8crzvqvw \
            --discovery-token-ca-cert-hash sha256:5098ee61994d4bbcf48746402e05d8772ac60d8160c46ddd5485721879e7a8a5
        ```
    - 安装pod网络
        ```
        # kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
        ```
    - 在master上管理集群（可选）
        ```
        # mkdir -p $HOME/.kube
        # sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        # sudo chown $(id -u):$(id -g) $HOME/.kube/config
        ```
    - 将master节点作为worker节点（可选）
        ```
        # kubectl taint nodes --all node-role.kubernetes.io/master-
        ```
3. Worker节点配置（node1、node2）
    - 加入master节点
        ```
        # kubeadm join 10.0.6.1:6443 --token fmot1q.7oicgu8h8crzvqvw \
                    --discovery-token-ca-cert-hash sha256:5098ee61994d4bbcf48746402e05d8772ac60d8160c46ddd5485721879e7a8a5
        ```
4. 部署dashboard
    - 下载官方 dashboard ymal 文件
        ```
        # wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
        ```
    - 修改 ymal 112 行为
        ```
        #image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1
        ```
    - 部署dashboard
        ```
        # kubectl apply -f kubernetes-dashboard.yaml
        ```
    - 部署集群服务账号
        ```
        # kubectl apply -f kubernetes-admin.yaml
        ```
    - 获取登录token
        ```
        # kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
        ```
    - 本地网络访问dashboard
        ```
        $ kubectl proxy
        
        http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
        ```
5. 部署 helm
    - 安装 helm 客户端（所有机器）
        ```
        1. 官方脚本安装（有墙）
        # curl https://raw.githubusercontent.com/helm/helm/master/scripts/get > get_helm.sh
        # chmod 700 get_helm.sh
        # ./get_helm.sh
        
        2. 官网二进制安装 https://github.com/kubernetes/helm/releases
        # tar -zxvf helm-v2.13.1-linux-amd64.tar.gz
        # mv linux-amd64/helm /usr/local/bin/helm
        ```
    - 安装 tiller 服务端（master）
        ```
        --- socat 要在所有机器上安装
        # yum install -y socat
        ```
        ```
        # helm init --client-only --stable-repo-url https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts/
        # helm repo add incubator https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts-incubator/
        # helm repo update
        
        -- 创建服务端
        # helm init --service-account tiller --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.13.1 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
        -- 创建TLS认证服务端，参考地址：https://github.com/gjmzj/kubeasz/blob/master/docs/guide/helm.md
        # helm init --service-account tiller --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.13.1 --tiller-tls-cert /etc/kubernetes/ssl/tiller001.pem --tiller-tls-key /etc/kubernetes/ssl/tiller001-key.pem --tls-ca-cert /etc/kubernetes/ssl/ca.pem --tiller-namespace kube-system --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
        ```
    - tiller 授权
        ```
        # kubectl create serviceaccount --namespace kube-system tiller
        # kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
        # kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
        ```