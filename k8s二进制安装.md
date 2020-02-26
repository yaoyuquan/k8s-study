# 通过二进制安装kubernates环境

## 1. 环境准备

  

本文档根据kubernates二进制安装包安装kubernates集群环境，需要五台centos7服务器，其中一台做为kubernates master主机名为master，一台服务器做为docker registry以及代码打包编译，构建镜像的环境服务器，其它三台服务器做为node服务器，另外一台centos服务器做为镜像服务器，运行docker registry



| 主机名         | IP  | 安装组件     |
|:-------------:|:---:|:-----------:|
| master        | 172.16.229.138 |kube-apiserver，kube-controller-manager，kube-scheduler             |
|node-a  | 172.16.229.140 | kubelet，kube-proxy，docker |
|node-b  | 172.16.229.143 | kubelet，kube-proxy，docker |
|node-c  | 172.16.229.142 | kubelet，kube-proxy，docker |
|docker-registry | 172.16.229.141 | docker，java，git，maven |



## 2. 安装前准备

在master和node上执行一下操作

1. 关闭防火墙
```
systemctl disable firewalld
systemctl stop firewalld
```
2. 下载kubernates二进制安装文件
```
wget https://github.com/kubernetes/kubernetes/releases/download/v1.17.3/kubernetes.tar.gz
```
3. 解压
```
tar -xzvf kubernetes.tar.gz
```
4. 运行脚本下载kubernates二进制执行文件，脚本目录在/cluster/get-kube-binaries.sh。（这一步可能需要通过代理访问）

   这个脚本会在/server下面下载压缩包kubernetes-server-linux-amd64.tar.gz，解压这个压缩包
   
```
tar -xzvf kubernetes-server-linux-amd64.tar.gz
```
解压缩后的文件说明  

| 文件名 | 说明 |
|:------|:---:|
|kube-apiserver|apiserver主程序|
|kube-apiserver.docker_tag|apiserver docker镜像的tag|
|kube-apiserver.tar|apiserver docker镜像文件|
|kube-controller-manager|controller-manager主程序|
|kube-controller-manager.docker_tag|controller-manager docker镜像的tag|
|kube-controller-manager.tar|controller-manager docker镜像文件|
|kubectl|客户端命令行工具|
|kubelet|kubelet主程序|
|kube-proxy|proxy主程序|
|kube-scheduler|scheduler主程序|
|kube-scheduler.docker_tag|kube-scheduler的docker镜像tag|
|kube-scheduler.tar|kube-scheduler的docker镜像文件|

5. master拷贝可执行文件kube-apiserver，kube-controller-manager，kube-scheduler，kubectl的可执行文件到/usr/bin

   node拷贝kubelet和kube-proxy可执行文件到/usr/bin
```
cp kube-apiserver /usr/bin
cp kube-controller-manager /usr/bin
cp kube-scheduler /usr/bin/
cp kubectl /usr/bin/

cp kubelet /usr/bin/
cp kube-proxy /usr/bin/
```


## 2. 安装master

### 2.1 安装etcd
etcd 服务 作为 Kubernetes 集群 的 主 数据库， 在 安装 Kubernetes 各 服务 之前 需要 首先 安装 和 启动。
```
yum install etcd
systemctl enable etcd
systemctl start etcd
```
查看etcd集群状态
```
etcdctl cluster-health
```

### 2.2 kube-apiserver服务

#### 2.2.1 kube-apiserver服务文件

在/usr/lib/systemd/system/创建文件kube-apiserver.service

```
[Unit] 
Description=Kubernetes API Server 
Documentation=https://github.com/GoogleCloudPlatform/kubernetes 
After=etcd.service 
Wants=etcd.service

[Service]
EnvironmentFile=/etc/kubernetes/apiserver
ExecStart=/usr/bin/kube-apiserver \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBE_ETCD_SERVERS \
	    $KUBE_API_ADDRESS \
	    $KUBE_API_PORT \
	    $KUBELET_PORT \
	    $KUBE_ALLOW_PRIV \
	    $KUBE_SERVICE_ADDRESSES \
	    $KUBE_ADMISSION_CONTROL \
	    $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```




#### 2.2.2 kube-apiserver环境配置
在/etc/kubernetes/创建环境参数文件apiserver
```
###
# kubernetes system config
#
# The following values are used to configure the kube-apiserver
#

# The address on the local server to listen to.
# 用于监听 --insecure-port 的 IP 地址 ( 设置成 0.0.0.0 表示监听所有接口 )。（默认值 127.0.0.1)
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"

# The port on the local server to listen on.
# 用于监听不安全和为认证访问的端口。
# 这个配置假设你已经设置了防火墙规则，使得这个端口不能从集群外访问。
# 对集群的公共地址的 443 端口的访问将被代理到这个端口。默认设置中使用 nginx 实现。（默认值 8080）
# KUBE_API_PORT="--insecure-port=8080"

# Port minions listen on
# KUBELET_PORT="--kubelet-port=10250"

# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://127.0.0.1:2379"

# Address range to use for services
# CIDR 表示的 IP 范围，服务的 cluster ip 将从中分配。 一定不要和分配给 nodes 和 pods 的 IP 范围产生重叠。
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# default admission control policies
# 控制资源进入集群的准入控制插件的顺序列表。逗号分隔的 NamespaceLifecycle 列表。（默认值 [AlwaysAdmit]）
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"

# Add your own!
KUBE_API_ARGS=""
```
有关kube-apiserver的详细环境参数可以参考
https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/

### 2.3 kube-controller-manager服务  
kube-controller-manager服务依赖于kube-apiserver服务。
Kubernetes 控制器管理器是一个守护进程，嵌入了 Kubernetes 附带的核心控制循环。 在机器人和自动化的应用中，控制回路是一个永不休止的循环，用于调节系统状态。 在 Kubernetes 中，控制器是一个控制循环，它通过 apiserver 监视集群的共享状态，并尝试进行更改以将当前状态转为所需状态。现今，Kubernetes 自带的控制器包括副本控制器，节点控制器，命名空间控制器和serviceaccounts 控制器。
#### 2.3.1 kube-controller-manager服务文件
在/usr/lib/systemd/system/创建文件kube-controller-manager.service
```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service

[Service]
EnvironmentFile=/etc/kubernetes/controller-manager
ExecStart=/usr/bin/kube-controller-manager \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBE_MASTER \
	    $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
#### 2.3.2 kube-controller-manager环境配置

在/etc/kubernetes/创建环境参数文件controller-manager

```
###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!
# logtostderr:log to standard error instead of files
# log-dir:If non-empty, write log files in this directory
# master:The address of the Kubernetes API server (overrides any value in kubeconfig).
KUBE_CONTROLLER_MANAGER_ARGS="--master=http://127.0.0.1:8080 --logtostderr=false --log-dir=/var/log/kubernetes/"
```
有关kube-controller-manager的详细环境参数可以参考
https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-controller-manager/

### 2.4 kube-scheduler服务

kube-scheduler服务也依赖于kube-apiserver服务

#### 2.4.1 kube-scheduler服务文件
在/usr/lib/systemd/system/创建文件kube-scheduler.service

```
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=/etc/kubernetes/scheduler
ExecStart=/usr/bin/kube-scheduler \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBE_MASTER \
	    $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```
#### 2.4.2 kube-scheduler环境配置
在/etc/kubernetes/创建环境参数文件scheduler
```
###
# kubernetes scheduler config

# default config should be adequate

# Add your own!
# logtostderr:log to standard error instead of files
# log-dir:If non-empty, write log files in this directory
# master:The address of the Kubernetes API server (overrides any value in kubeconfig).
KUBE_SCHEDULER_ARGS="--master=http://127.0.0.1:8080 --logtostderr=false --log-dir=/var/log/kubernetes/"
```
有关kube-scheduler的详细环境参数可以参考
https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-scheduler/

### 2.5 启动服务
```
systemctl daemon-reload
systemctl enable kube-apiserver.service
systemctl start kube-apiserver.service
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl enable kube-scheduler
systemctl start kube-scheduler
```
启动之后可以通过以下命令查看对应服务的状态
```
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
```
至此，master搭建完成



### 2.6 生成kubeconfig文件

这个步骤所用到的命令请参考
https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config
1. 创建用户，用户名为admin，密码1234
```
kubectl config set-credentials cluster-admin --username=admin --password=1234
```
2. 创建集群，名称为test-cluster
```
kubectl config set-cluster test-cluster --insecure-skip-tls-verify=true
```
3. 创建对应的context，名称为test-context
```
kubectl config set-context test-context --cluster=test-cluster --user=cluster-admin
```
4. 应用当前context为test-context
```
kubectl config use-context test-context
```
下载/root/.kube/config文件，将这个文件拷贝到node服务器的/etc/kubernetes中

修改文件中的clusters-cluster-server字段，修改为kube-apiserver的地址http://172.16.229.138:8080

## 3. 安装node

### 3.1 关闭swap
kubernates环境需要关闭linux swap，否则kubelet将无法和apiserver通信。
```
swapoff -a
```
然后修改/etc/fstab 注释掉swap一行，然后重启

### 3.2 安装docker

```
yum install yum-utils
yum install device-mapper-persistent-data
yum install lvm2

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install docker-ce docker-ce-cli containerd.io

systemctl enable docker
systemctl start docker
```
### 3.3 kubelet服务
#### 3.3.1 kubelet服务文件
在/usr/lib/systemd/system中创建文件kubelet.service
```
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBELET_API_SERVER \
	    $KUBELET_ADDRESS \
	    $KUBELET_PORT \
	    $KUBELET_HOSTNAME \
	    $KUBE_ALLOW_PRIV \
	    $KUBELET_POD_INFRA_CONTAINER \
	    $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
#### 3.3.2 kubelet环境配置 
在/etc/kubernetes/中编辑文件kubelet
```
###
# kubernetes kubelet (minion) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
# kubelet 的服务 IP 地址（所有 IPv4 接口设置为 0.0.0.0 ，所有 IPv6 接口设置为 “::”）（默认值为 0.0.0.0）
#（deprecated：在 --config 指定的配置文件中进行设置。有关更多信息，请参阅 https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/。）
KUBELET_ADDRESS="--address=0.0.0.0"

# The port for the info server to serve on
# KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
# 如果为非空，将使用此字符串而不是实际的主机名作为节点标识。
# 如果设置了 --cloud-provider，则云服务商将确定节点的名称（请查询云服务商文档以确定是否以及如何使用主机名）。
KUBELET_HOSTNAME="--hostname-override=node-a"

# pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"

# Add your own!
# kubeconfig:kubeconfig 配置文件的路径，指定如何连接到 API 服务器。提供 --kubeconfig 将启用 API 服务器模式，而省略 --kubeconfig 将启用独立模式。
# logtostderr:log to standard error instead of files
# log-dir:If non-empty, write log files in this directory
KUBELET_ARGS="--kubeconfig=/etc/kubernetes/config --logtostderr=false --log-dir=/var/log/kubernetes/"

```

有关kubelet的详细环境参数可以参考

https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/
#### 3.3.3 创建kubelet工作目录
在/var/lib下创建目录kubelet

### 3.4 kube-proxy服务
Kubernetes 网络代理在每个节点上运行。网络代理反映了每个节点上 Kubernetes API 中定义的服务，并且可以执行简单的 TCP、UDP 和 SCTP 流转发，或者在一组后端进行循环 TCP、UDP 和 SCTP 转发。当前可通过 Docker-links-compatible 环境变量找到服务集群 IP 和端口，这些环境变量指定了服务代理打开的端口。有一个可选的插件，可以为这些集群 IP 提供集群 DNS。用户必须使用 apiserver API 创建服务才能配置代理。

kube-proxy服务依赖于network服务
#### 3.4.1 kube-proxy服务文件
在/usr/lib/systemd/system下创建文件kube-proxy.service

```
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
Requires=network.service

[Service]
EnvironmentFile=/etc/kubernetes/proxy
ExecStart=/usr/bin/kube-proxy \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBE_MASTER \
	    $KUBE_PROXY_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
#### 3.4.2 kube-proxy环境配置
在/etc/kubernetes/下创建文件proxy
```
###
# kubernetes proxy config

# default config should be adequate

# Add your own!
# logtostderr:log to standard error instead of files
# log-dir:If non-empty, write log files in this directory
# kubeconfig:kubeconfig 配置文件的路径，指定如何连接到 API 服务器。
# 提供 --kubeconfig 将启用 API 服务器模式，而省略 --kubeconfig 将启用独立模式。
KUBE_PROXY_ARGS="--kubeconfig=/etc/kubernetes/config --logtostderr=false --log-dir=/var/log/kubernetes/"
```
有关kube-proxy的相关配置可以参考
https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-proxy/

### 3.5 启动服务

```
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl enable kube-proxy
systemctl start kube-proxy
```
最后用同样的方法配置node-a，node-b，node-c