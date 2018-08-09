# 使用K8S部署Selenium Grid集群

## 前言

[speedy](https://gitlab.com/AngryTester/speedy)是基于Selenium Grid+Docker的自动化测试框架，之前一直是将Grid集群中在公司的私有云平台上。虽然之前也用docker-compose搭建过Grid集群，但是并没有考虑多节点横向扩展的能力，而这恰恰就是K8S的强项。正好最近有一个微服务项目应用到K8S，借此机会学习了一把K8S的搭建和简单使用，正好拿Grid集群的搭建作为例子，在此做个记录。

### 搭建

> 环境准备

通过VirtualBox准备了三台centos，分别为：
```
172.26.X.60-master

172.26.X.61-minion1

172.26.X.62-minion2
```
虚拟机镜像选择`CentOS-7-x86_64-Minimal-1804.iso`


#### 1.虚拟机启动之后更新yum源

分别执行：

```
yum update -y
```

#### 2.配置三台服务器的Host

分别执行：

```
echo -e "172.26.X.60 master\n\
172.26.X.61 minion1\n\
172.26.X.62 minion2" >> /etc/hosts
```

#### 3.关闭防火墙和selinux

分别执行：

```
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

#### 4.修改网桥设置

```
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.d/k8s.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.d/k8s.conf
echo "#关闭需虚拟内存"
echo "vm.swappiness=0" >> /etc/sysctl.d/k8s.conf
sysctl -p /etc/sysctl.d/k8s.conf
```

    centos7添加bridge-nf-call-ip6tables出现No such file or directory

    解决方法：

    [root@localhost ~]# modprobe br_netfilter
    [root@localhost ~]# ls /proc/sys/net/bridge
    bridge-nf-call-arptables bridge-nf-filter-pppoe-tagged
    bridge-nf-call-ip6tables bridge-nf-filter-vlan-tagged
    bridge-nf-call-iptables bridge-nf-pass-vlan-input-dev
 

#### 5.关闭swap

分别执行：

```
sed -i "s/^.*swap.*/#&/g" /etc/fstab
```
重启服务器。

#### 6.配置master和minion的免密互联

分别执行：
```
ssh-keygen -t rsa
```
一路回车。

分别执行：
```
cat ~/.ssh/id_rsa.pub > ~/.ssh/authorized_keys
```
master上执行：
```
ssh-copy-id minion1&&ssh-copy-id minion2
```

minion上分别执行：
```
ssh-copy-id master
```

#### 7.安装docker

可选择官方提供的安装方式，也可使用离线安装，使用的安装文件为：`docker-ce-17.03.2.ce-1.el7.centos.x86_64.rpm`和`docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm`

可手动下载离线安装包：

```
yum install --downloadonly --downloaddir=/home  docker-ce-17.03.2.ce docker-ce-selinux-17.03.2.ce
```

分别执行：

```
yum localinstall docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm
yum localinstall docker-ce-17.03.2.ce-1.el7.centos.x86_64.rpm
```

Docker从1.13版本开始调整了默认的防火墙规则，禁用了iptables filter表中FOWARD链，这样会引起Kubernetes集群中跨Node的Pod无法通信，在各个Docker节点执行下面的命令：

配置docker的策略

```
vi /usr/lib/systemd/system/docker.service
#ExecStart=上面加一行：
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT
```

启动docker：
 ```
systemctl enable docker
systemctl start docker
 ```

#### 8.导入kubeadm、kubelet、kubectl

可手动下载离线安装包：
```
yum install --downloadonly --downloaddir=/home/k8s196  kubelet-1.9.6-0  kubectl-1.9.6-0 kubeadm-1.9.6-0
```

分别执行：
```
yum localinstall  *
```

#### 9.导入相关镜像

需要准备镜像如下：
```
busybox.tar
etcd-amd64_3.1.11.tar
flannel-0.9.1-amd64.tar
k8s-dns-dnsmasq-nanny-amd64_v1.14.7.tar 
k8s-dns-kube-dns-amd64_1.14.7.tar
k8s-dns-sidecar-amd64_1.14.7.tar
kube-apiserver-amd64_v1.9.6.tar
kube-controller-manager-amd64_v1.9.6.tar
kube-proxy-amd64_v1.9.6.tar
kubernetes-dashboard-amd64_v1.8.3.tar
kube-scheduler-amd64_v1.9.6.tar
pause-amd64_3.0.tar
hub.tar
chrome.tar
registry.tar
```

在镜像文件目录新建文件`images.txt`,文件内容如下:
```
busybox.tar
etcd-amd64_3.1.11.tar
flannel-0.9.1-amd64.tar
k8s-dns-dnsmasq-nanny-amd64_v1.14.7.tar 
k8s-dns-kube-dns-amd64_1.14.7.tar
k8s-dns-sidecar-amd64_1.14.7.tar
kube-apiserver-amd64_v1.9.6.tar
kube-controller-manager-amd64_v1.9.6.tar
kube-proxy-amd64_v1.9.6.tar
kubernetes-dashboard-amd64_v1.8.3.tar
kube-scheduler-amd64_v1.9.6.tar
pause-amd64_3.0.tar
hub.tar
chrome.tar
registry.tar
```

在镜像文件目录执行：

```
for i in `cat images.txt ` ; do docker load < `echo $i |cut -d '/' -f 3` ; done
# docker images
REPOSITORY                                                   TAG                 IMAGE ID            CREATED             SIZE
registry                                                     latest              b2b03e9146e1        4 weeks ago         33.3 MB
registry.gitlab.com/angrytester/selenium-hub                 3.4                 13febb5e0aa9        2 months ago        293 MB
registry.gitlab.com/angrytester/selenium-node-chrome-debug   3.4                 ea211cd77500        2 months ago        989 MB
busybox                                                      latest              8c811b4aec35        2 months ago        1.15 MB
gcr.io/google_containers/kube-controller-manager-amd64       v1.9.6              472b6fcfe871        4 months ago        139 MB
gcr.io/google_containers/kube-apiserver-amd64                v1.9.6              a5c066e8c9bf        4 months ago        212 MB
gcr.io/google_containers/kube-proxy-amd64                    v1.9.6              70e63dd90b80        4 months ago        109 MB
gcr.io/google_containers/kube-scheduler-amd64                v1.9.6              25d7b2c6f653        4 months ago        62.9 MB
gcr.io/google_containers/kubernetes-dashboard-amd64          v1.8.3              0c60bcf89900        5 months ago        102 MB
gcr.io/google_containers/etcd-amd64                          3.1.11              59d36f27cceb        8 months ago        194 MB
quay.io/coreos/flannel                                       v0.9.1-amd64        2b736d06ca4c        8 months ago        51.3 MB
gcr.io/google_containers/k8s-dns-sidecar-amd64               1.14.7              db76ee297b85        9 months ago        42 MB
gcr.io/google_containers/k8s-dns-kube-dns-amd64              1.14.7              5d049a8c4eec        9 months ago        50.3 MB
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64         1.14.7              5feec37454f4        9 months ago        40.9 MB
gcr.io/google_containers/pause-amd64                         3.0                 99e59f495ffa        2 years ago         747 kB
```

#### 10.搭建本地镜像仓库

由于后续需要拉取镜像，避免直接从公网拉取，在master节点上搭建本地镜像仓库（也可搭建在其他局域网服务器上），启动master节点的docker服务：
```
systemctl enable docker
systemctl start docker
```

启动一个registry容器：
```
docker run -d -p 5000:5000 --restart=always registry 
```
分别修改三台服务器的docker启动配置后重启docker：
```
echo {\"insecure-registries\": [\"172.26.X.60:5000\"]} > /etc/docker/daemon.json
systemctl daemon-reload
systemctl restart docker
```

#### 11.将pause-amd64上传镜像库

修改镜像tag：
```
docker tag gcr.io/google_containers/pause-amd64:3.0 172.26.X.60:5000/pause-amd64:3.0
docker push 172.26.X.60:5000/pause-amd64:3.0
```

#### 12.设置进程隔离方式以及启动参数

由于docker默认的隔离方式为cgroup,而k8s默认的是systemd,需要改为一致

```
sed -i 's/systemd/cgroupfs/g' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```


```
echo Environment=\"KUBELET_SWAP_ARGS=--fail-swap-on=false\" >> /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
echo Environment=\"KUBELET_INFRA_IMAGE=--pod-infra-container-image=172.26.X.60:5000/pause-amd64:3.0\" >> /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

#### 13.启动K8S

分别执行:

```
systemctl daemon-reload
systemctl enable kubelet.service
systemctl start kubelet.service
```

### 初始化集群

```
kubeadm init --kubernetes-version=v1.9.6 --pod-network-cidr=10.244.0.0/16 --token-ttl=0
```

`--token-ttl=0`表示token永不过期

最后会返回类似如下：
```
kubeadm join --token 1ec757.1865c26203561363 172.26.X.60:6443 --discovery-token-ca-cert-hash sha256:004d45cc446b613a054451c680b1506145987b9b53c6290fa7040eb40e01ef7f
```

在两个minion上分别执行如上命令加入集群。

可通过`docker ps -a`查看master节点和minion节点上容器的启动情况：

master节点上应该启动5个pause容器以及`kube-proxy`,`kube-controller`,`kube-scheduler`,`kube-apiserver`和`etcd`。

minion节点上应该启动1个pause容器以及`kube-proxy`。

K8S上所有pod的启动都会对应启动一个pause容器，主要作用是为了使pod里面的容器共享pause容器的网络。


### 配置kubectl命令执行权限

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
如果需要在minion节点上执行kubectl命令，也需要将`admin.conf`复制到minion上。

通过如下命令获取pod信息：
```
kubectl  get pods --all-namespaces
kube-system   etcd-master                      1/1       Running   0          2s
kube-system   kube-apiserver-master            1/1       Running   0          2s
kube-system   kube-controller-manager-master   1/1       Running   0          2s
kube-system   kube-dns-6f4fd4bdf-x27zx         0/3       Pending   0          9m
kube-system   kube-proxy-kdkj2                 1/1       Running   0          9m
kube-system   kube-scheduler-master            1/1       Running   0          2s
```

### 将KUBECONFIG写入环境变量
```
echo KUBECONFIG=/etc/kubernetes/admin.conf >> ~/.bash_profile
source ~/.bash_profile
```

### 创建flannel

在未创建flannel之前，kube-dns状态为pending。

执行如下两句命令创建flannel：
```
kubectl apply -f kube-flannel-rbac.yaml  
kubectl apply -f  kube-flannel.yaml
```

`kube-flannel-rbac.yaml`文件[参考](https://raw.githubusercontent.com/AngryTester/K8S_Selenium_Grid/master/kube-flannel-rbac.yaml)

`kube-flannel.yaml`文件[参考](https://raw.githubusercontent.com/AngryTester/K8S_Selenium_Grid/master/kube-flannel.yml)

### 创建dashboard

执行如下命令创建dashboard：
```
kubectl apply -f kubernetes-dashboard.yaml
```
`kubernetes-dashboard.yaml`文件[参考](https://raw.githubusercontent.com/AngryTester/K8S_Selenium_Grid/master/kubernetes-dashboard.yaml)

### 创建dashboard认证

```
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```
增加如下内容：
```
- --basic-auth-file=/etc/kubernetes/pki/basic_auth_file
```

`/etc/kubernetes/pki/basic_auth_file`文件内容如下：
```
admin,admin,2
```

重启kubelet：
```
systemctl daemon-reload
systemctl restart kubelet
```

然后创建账号和角色的关联
```
kubectl create clusterrolebinding login-on-dashboard-with-cluster-admin --clusterrole=cluster-admin --user=admin
```

访问 https://172.26.X.60:30000 ,输入用户名`admin`和密码`admin`即可登录dashboard。

另外一种方式token，首先需要将`kubernetes-dashboard.yaml`中验证方式由`basic`修改为token。

token 可以通过相关指令获取：
```
kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token
```

 不同类型的token关联的权限是不一样的
 ```
# 输入下面命令查询kube-system命名空间下的所有secret，然后获取token。只要是type为service-account-token的secret的token都可以使用
kubectl get secret -n kube-system
# 比如我们获取replicaset-controller-token-wsv4v的touken
kubectl -n kube-system describe replicaset-controller-token-wsv4v
```

### 创建selenium-hub

`selenium.namespace.yml`内容[参考](https://raw.githubusercontent.com/Polygon-io/selenium-on-kubernetes/master/selenium.namespace.yml)
`selenium-hub.deployment.yml`内容[参考](https://raw.githubusercontent.com/Polygon-io/selenium-on-kubernetes/master/selenium-hub.deployment.yml)
`selenium-hub.service.yml`内容[参考](https://raw.githubusercontent.com/Polygon-io/selenium-on-kubernetes/master/selenium-hub.service.yml)

```
kubectl apply -f selenium.namespace.yml
kubectl apply -f selenium-hub.deployment.yml
kubectl apply -f selenium-hub.service.yml
```

### 创建selenium-node

`selenium-chrome.deployment.yml`内容[参考](https://raw.githubusercontent.com/Polygon-io/selenium-on-kubernetes/master/selenium-chrome.deployment.yml)

```
kubectl apply -f selenium-chrome.deployment.yml
```

扩展node可参考：

```
kubectl scale --replicas=3 -f selenium-chrome.deployment.yml
```

访问http://172.26.X.60:30001/grid/console  即能看到Grid集群情况。







