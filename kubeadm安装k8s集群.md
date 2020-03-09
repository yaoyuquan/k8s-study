# 使用kubeadm建立k8s集群

## 系统需求
* Centos7

* 2核CPU

* 4G内存

* 20G磁盘空间
## 安装前准备
### 关闭swap
kubelet运行需要关闭swap
```
swapoff -a
```
然后修改/etc/fstab 注释掉swap一行，然后重启

### 设置iptable
创建文件/etc/sysctl.d/k8s.conf
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
运行命令 sysctl --system

### 关闭防火墙
```
systemctl disable firewalld
systemctl stop firewalld
```
### 修改hostname
k8s集群要求每台主机的hostname不能重复，所以需要分别将每台主机的hostname改成master，node-a，node-b，node-c
修改/etc/hostname ， 修改/etc/hosts完成后重启




## 安装docker
1. 安装yum-utils
```
yum install -y yum-utils device-mapper-persistent-data lvm2
```
2. 安装docker的yum仓库
```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
3. 安装docker
```
yum install docker-ce docekr-ce-cli containerd.io
```
4. 启动docker，并且配置docker开机启动
```
systemctl start docker
systemctl enable docker
```





## yum安装kubeadm
创建文件/etc/yum.repos.d/kubernetes.repo
```
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```
安装
```
# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet
```


## 初始化cluster
执行kubeadm init命令初始化集群，命令如果执行成功，可以看到下面的提示，需要保存下面的文本，将其他节点加入到集群会用到下面的命令。
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

kubeadm join 172.16.229.138:6443 --token ws7g7g.bm790385bzefeu9z \
    --discovery-token-ca-cert-hash sha256:774980072fed36a6b5546d4a67f707e50eddf4627ed0e3c59c4931f929c76226


```

如果是root用户，可以设置环境变量，设置成功后，可以运行kubectl
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```