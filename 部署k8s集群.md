[TOC]

### 基本环境配置

1. 关闭selinux

   ```sh
   setenforce 0
   sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
   ```

2. 关闭swap分区或禁用swap文件

   ```sh
   swapoff -a
   # 注释掉关于swap分区的行
   vim /etc/fstab
   #/dev/mapper/centos-swap swap                    swap    defaults        0 0
   ```

3. 修改网卡配置

   ```sh
   $ vim /etc/sysctl.conf
   net.ipv4.ip_forward = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   $ sysctl -p
   
   
   # 配置转发相关参数，否则可能会出错
   cat <<EOF > /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.conf.all.forwarding = 1
   vm.swappiness = 0
   EOF
   # 使配置生效，查看输出中的相关参数是否被其它文件覆盖
   sysctl --system
   # 为防止被覆盖，可执行下面的命令
   sysctl -p /etc/sysctl.d/k8s.conf
   ```

4. 启用内核模块

   ```sh
   $ modprobe -- ip_vs
   $ modprobe -- ip_vs_rr
   $ modprobe -- ip_vs_wrr
   $ modprobe -- ip_vs_sh
   $ modprobe -- nf_conntrack_ipv4
   $ cut -f1 -d " "  /proc/modules | grep -e ip_vs -e nf_conntrack_ipv4
   $ vim /etc/sysconfig/modules/ipvs.modules
   modprobe -- ip_vs
   modprobe -- ip_vs_rr
   modprobe -- ip_vs_wrr
   modprobe -- ip_vs_sh
   modprobe -- nf_conntrack_ipv4
   ```

5. 关闭防火墙

   ```sh
   $ systemctl stop firewalld
   $ systemctl disable firewalld
   ```

6. 配置hosts

   ```sh
   hostnamectl set-hostname kube-master
   ```

### kubectl、kubeadm、kubelet的安装

#### 添加Kubernetes的yum源

此处使用alibaba的镜像源

```sh
$vim /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
	http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```



#### 安装kubelet、kubeadm、kubectl

```sh
$ yum install -y kubelet kubeadm kubectl
```

#### 启动kubelet服务

```sh
$ systemctl enable kubelet
$ systemctl start kubelet
```

此时执行`systemctl status kubelet`查看服务状态，服务状态应为Error(255)， 如果是其他错误可使用`journalctl -xe`查看错误信息。

### Docker安装和配置

#### Docker安装

docker的安装请查看官网文档(Overview of Docker editions)[https://docs.docker.com/install/overview/]

#### Docker配置

1. 配置cgroup-driver为systemd

   ```sh
   # 查看cgroup-driver
   $ docker info | grep -i cgroup
   # 修改docker配置文件
   $ vim /etc/docker/daemon.json
   {
     "exec-opts": ["native.cgroupdriver=systemd"]
   }
   $ systemctl daemon-reload # 重新加载服务
   $ systemctl enable docker # 启用docker服务(开机自起)
   $ systemctl restart docker # 启动docker服务
   ```

2. 预先拉取所需镜像

   ```shell
   # 查看kubeadm所需镜像
   $ kubeadm config images list
   k8s.gcr.io/kube-apiserver:v1.20.1
   k8s.gcr.io/kube-controller-manager:v1.20.1
   k8s.gcr.io/kube-scheduler:v1.20.1
   k8s.gcr.io/kube-proxy:v1.20.1
   k8s.gcr.io/pause:3.2
   k8s.gcr.io/etcd:3.4.13-0
   k8s.gcr.io/coredns:1.7.0
   # 拉取镜像
   $ docker pull kubeimage/kube-apiserver-amd64:v1.20.1
   $ docker pull kubeimage/kube-controller-manager-amd64:v1.20.1
   $ docker pull kubeimage/kube-scheduler-amd64:v1.20.1
   $ docker pull kubeimage/kube-proxy-amd64:v1.20.1
   $ docker pull kubeimage/pause-amd64:3.2
   $ docker pull kubeimage/etcd-amd64:3.4.13-0
   $ docker pull coredns/coredns:1.7.0
   
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.1
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.1
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.1
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.1
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
   docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0
   
   ```

3. 对预先拉取的镜像重新打tag

   ```shell
   $ docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.1  k8s.gcr.io/kube-apiserver:v1.20.1
   $ docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.1 k8s.gcr.io/kube-controller-manager:v1.20.1
   $ docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.1  k8s.gcr.io/kube-scheduler:v1.20.1
   $ docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.1 k8s.gcr.io/kube-proxy:v1.20.1
   $ docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
   $ docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0
   $ docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0
   ```

### Master节点的配置

***以上步骤需要在node节点和master节点执行，当前步骤仅需在master节点执行。\***

#### Master节点的初始化

```sh
# 初始化master节点，
# --pod-network-cidr=192.168.0.0/16 指定使用flannel网络
# --apiserver-advertise-address=10.0.0.5 指向master节点IP，此处也可以使用hosts
$ kubeadm init --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=v1.20.1 \
  --apiserver-advertise-address=129.28.162.62
  
失败后 kubeadm reset
```

执行上述命令的输出为：

```
[init] Using Kubernetes version: v1.16.3
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 19.03.4. Latest validated version: 18.09
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.0.5]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [master localhost] and IPs [10.0.0.5 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [master localhost] and IPs [10.0.0.5 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 13.002108 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.16" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: kt58np.djd3youoqb0bnz4r
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.5:6443 --token kt58np.djd3youoqb0bnz4r \
    --discovery-token-ca-cert-hash sha256:37a3924142dc6d57eac2714e539c174ee3b0cda723746ada2464ac9e8a2091ce
```



保存输出中的`kubeadm join`部分内容，用于添加node节点，或者使用`kubeadm token list` 和`kubeadm token create --print-join-command`查看

```sh
$ kubeadm join 10.0.0.5:6443 --token kt58np.djd3youoqb0bnz4r \
		--discovery-token-ca-cert-hash sha256:37a3924142dc6d57eac2714e539c174ee3b0cda723746ada2464ac9e8a2091ce
```

接下来执行剩余的初始化步骤

```sh
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



#### Flannel网络插件的配置

1. 安装flannel

   ```sh
   $ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   ```

2. 监听安装进度

   ```sh
   $ watch kubectl get pods --all-namespaces
   ```

出现以下内容时为安装成功

```sh
NAMESPACE    NAME                                       READY  STATUS   RESTARTS  AGE
kube-system  calico-kube-controllers-6ff88bf6d4-tgtzb   1/1    Running  0         2m45s
kube-system  calico-node-24h85                          1/1    Running  0         2m43s
kube-system  coredns-846jhw23g9-9af73                   1/1    Running  0         4m5s
kube-system  coredns-846jhw23g9-hmswk                   1/1    Running  0         4m5s
kube-system  etcd-jbaker-1                              1/1    Running  0         6m22s
kube-system  kube-apiserver-jbaker-1                    1/1    Running  0         6m12s
kube-system  kube-controller-manager-jbaker-1           1/1    Running  0         6m16s
kube-system  kube-proxy-8fzp2                           1/1    Running  0         5m16s
kube-system  kube-scheduler-jbaker-1                    1/1    Running  0         5m41s
```



1. 测试

   ```sh
   $ kubectl get nodes -o wide
   NAME                STATUS     ROLES    AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
   kubernetes-master   Ready      master   4d12h   v1.16.3   192.168.56.101   <none>        CentOS Linux 7 (Core)   3.10.0-1062.el7.x86_64   docker://19.3.4
   ```

### Node节点的初始化

1. 登录node节点，执行加入集群的命令，完成加入集群操作

   ```sh
   $ kubeadm join 10.0.0.5:6443 --token kt58np.djd3youoqb0bnz4r \
       --discovery-token-ca-cert-hash sha256:37a3924142dc6d57eac2714e539c174ee3b0cda723746ada2464ac9e8a2091ce
   ```

2. 在master节点上查看添加结果

   ```
   $ kubectl get nodes -o wide
   NAME                STATUS     ROLES    AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
   kubernetes-master   Ready      master   4d12h   v1.16.3   192.168.56.101   <none>        CentOS Linux 7 (Core)   3.10.0-1062.el7.x86_64   docker://19.3.4
   kubernetes-node-1   Ready      <none>   4d12h   v1.16.3   192.168.56.102   <none>        CentOS Linux 7 (Core)   3.10.0-1062.el7.x86_64   docker://19.3.4
   ```