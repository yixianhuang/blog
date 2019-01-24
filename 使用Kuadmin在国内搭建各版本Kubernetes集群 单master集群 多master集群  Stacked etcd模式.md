## 部署架构图
官网提供了2种高可用部署方式。区别就是etcd是否在容器里面部署。本人这次使用容器部署的方式。发现还是很多坑的。如在加入第二个master的时候失败了，第一个master的etcd起不来了，好像是修改了configmap，但是此时我的etcd起不来，又修改不了etcd的configmap。以后还是建议把etcd部署在外面的方式。
感兴趣的朋友可以查阅官网：https://kubernetes.io/docs/setup/independent/ha-topology/
架构图如下：

![avatar](https://d33wubrfki0l68.cloudfront.net/d1411cded83856552f37911eb4522d9887ca4e83/b94b2/images/kubeadm/kubeadm-ha-topology-stacked-etcd.svg)


## 服务器信息

| 主机名 | ip | 系统版本 |备注 |
| --- | --- | --- |--- |
| k8snode01 | 192.168.33.61 |CentOS Linux release 7.6.1810 (Core)| master etcd |
| k8snode02 | 192.168.33.62 |CentOS Linux release 7.6.1810 (Core)| master etcd |
| k8snode03 | 192.168.33.63 |CentOS Linux release 7.6.1810 (Core)| master etcd |
| k8snode04 | 192.168.33.64 |CentOS Linux release 7.6.1810 (Core)| work |
| k8snode05 | 192.168.33.65 |CentOS Linux release 7.6.1810 (Core)| work |
| vip | 192.168.33.66 || 虚拟机ip,如果是搭建单集群可以不使用，如果多集群使用LoadBalance也可以不使用 |

```bash
#参考信息
#查看系统版本信息
cat /etc/redhat-release
```

## 检查端口是否开通
### Master node(s)
| Protocol | Direction | Port Range | Purpose | Used By |
| --- | --- | --- | --- | --- |
| TCP | Inbound | 6443* | Kubernetes API server | All |
| TCP | Inbound | 2379-2380 | etcd server client API | kube-apiserver, etcd|
|TCP	|Inbound|	10250|	Kubelet API|	Self, Control plane|
|TCP	|Inbound|	10251|	kube-scheduler|	Self|
|TCP	|Inbound|	10252|	kube-controller-manager|	Self|

### Worker node(s)

|Protocol|	Direction|	Port Range	|Purpose	|Used By|
| --- | --- | --- | --- | --- |
|TCP |Inbound| 10250| Kubelet API| Self, Control plane |
|TCP |Inbound| 30000-32767|	NodePort Services**	|All|


## 软件版本
```bash
docker17.03.2-ce
socat-1.7.3.2-2.el7.x86_64
kubelet-1.10.0-0.x86_64
kubernetes-cni-0.6.0-0.x86_64
kubectl-1.10.0-0.x86_64
kubeadm-1.10.0-0.x86_64
```
## 环境初始化（root用户执行）
###  1.每台服务上修改主机名
```bash
#修改第1台服务器
hostnamectl set-hostname k8snode01
#修改第2台服务器
hostnamectl set-hostname k8snode02
#修改第3台服务器
hostnamectl set-hostname k8snode03
#修改第4台服务器
hostnamectl set-hostname k8snode04
#修改第5台服务器
hostnamectl set-hostname k8snode05
```

### 2.每台服务器上配置host
```bash
cat <<EOF >> /etc/hosts
192.168.33.61 k8snode01
192.168.33.62 k8snode02
192.168.33.63 k8snode03
192.168.33.64 k8snode04
192.168.33.65 k8snode05
EOF
```

### 3.每台服务器进行配置、停防火墙、关闭Swap、关闭Selinux、设置内核、K8S的yum源、安装依赖包、配置ntp（配置完后建议重启一次）
```bash
#关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
#关闭Swap
swapoff -a 
sed -i 's/.*swap.*/#&/' /etc/fstab
#关闭Selinux
setenforce  0 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config  
#要求iptables对bridge的数据进行处理
modprobe br_netfilter
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf
ls /proc/sys/net/bridge
#配置yum为阿里源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
#安装基础包
yum install -y epel-release
yum install -y yum-utils device-mapper-persistent-data lvm2 net-tools conntrack-tools wget vim  ntpdate libseccomp libtool-ltdl 
#设置文件最大链接数 有助于提高性能
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf
```
```bash
#设置定时同步时间（二者选择其一，本人使用第1种方式）
#1.采用定时任务的形式同步时间
systemctl enable ntpdate.service
echo '*/30 * * * * /usr/sbin/ntpdate time7.aliyun.com >/dev/null 2>&1' > /tmp/crontab2.tmp
crontab /tmp/crontab2.tmp
systemctl start ntpdate.service
```
```bash
#2.通过修改配置文件修改时区，然后使用ntp同步时间
#参考 https://choerodon.io/zh/docs/installation-configuration/steps/kubernetes/
1、远程连接 Linux 服务器。
2、执行命令 sudo rm /etc/localtime 删除系统里的当地时间链接。
3、执行命令 sudo vi /etc/sysconfig/clock 用 vim 打开并编辑配置文件 /etc/sysconfig/clock。
4、输入 i 添加时区城市，例如添加 Zone=Asia/Shanghai，按下 Esc 键退出编辑并输入 :wq 保存并退出。（可执行命令 ls /usr/share/zoneinfo 查询时区列表，Shanghai 为列表条目之一。）
5、执行命令 sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai/etc/localtime 更新6、时区修改内容。
7、执行命令 hwclock -w 更新硬件时钟（RTC）。
8、执行命令 sudo reboot 重启实例。
9、执行命令 date -R 查看时区信息是否生效，未生效可重走一遍步骤。
10、安装ntp服务sudo yum install ntp。
11、修改成国内时区并同步。
timedatectl set-timezone Asia/Shanghai
timedatectl set-ntp yes
```
```bash
#修改ssh文件,方便后续在k8snode01使用scp命令传输文件都其他节点上
vi /etc/ssh/sshd_config
#把下面的属性设置为yes
PasswordAuthentication yes
PubkeyAuthentication yes
PermitRootLogin yes
#重启ssh服务
sudo systemctl restart sshd.service
#查看ssh服务的状态
sudo systemctl status sshd.service

```

```bash
#在k8snode01上执行ssh免密码登陆配置，方便后续scp命令传输文件
ssh-keygen  #一路回车即可
ssh-copy-id  k8snode02
ssh-copy-id  k8snode03
ssh-copy-id  k8snode04
ssh-copy-id  k8snode05
```

```bash
#重启每个服务器
reboot
```

### 4.安装、配置keepalived（master节点）。单集群或使用LoadBalance请跳过。

```bash
#4.1 安装keepalived
yum install -y keepalived
systemctl enable keepalived
```

```bash
#k8snode01的keepalived.conf，以下地方注意修改

#{192.168.33.66:6443} 为虚拟机ip 6443端口为kubernetes api server 端口
#{interface eth1} 把虚拟ip绑定到网卡上，可使用命令 ip addr查看本机器在网卡上
#{mcast_src_ip 192.168.33.61} 为本节点ip地址
#{unicast_peer} 内容为其他master节点的ip
#{virtual_ipaddress} 为虚拟ip的地址

cat <<EOF > /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://192.168.33.66:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 61
    priority 100
    advert_int 1
    mcast_src_ip 192.168.33.61
    nopreempt
    authentication {
        auth_type PASS
        auth_pass sqP05dQgMSlzrxHj
    }
    unicast_peer {
        192.168.33.62
        192.168.33.63
    }
    virtual_ipaddress {
        192.168.33.66/24
    }
    track_script {
        CheckK8sMaster
    }

}
EOF
```

```bash
#k8snode02的keepalived.conf，以下地方注意修改

#{192.168.33.66:6443} 为虚拟机ip 6443端口为kubernetes api server 端口
#{interface eth1} 把虚拟ip绑定到网卡上，可使用命令 ip addr查看本机器在网卡上
#{mcast_src_ip 192.168.33.62} 为本节点ip地址
#{unicast_peer} 内容为其他master节点的ip
#{virtual_ipaddress} 为虚拟ip的地址

cat <<EOF > /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://192.168.33.66:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 61
    priority 90
    advert_int 1
    mcast_src_ip 192.168.33.62
    nopreempt
    authentication {
        auth_type PASS
        auth_pass sqP05dQgMSlzrxHj
    }
    unicast_peer {
        192.168.33.61
        192.168.33.63
    }
    virtual_ipaddress {
        192.168.33.66/24
    }
    track_script {
        CheckK8sMaster
    }

}
EOF
```

```bash
#k8snode02的keepalived.conf，以下地方注意修改

#{192.168.33.66:6443} 为虚拟机ip 6443端口为kubernetes api server 端口
#{interface eth1} 把虚拟ip绑定到网卡上，可使用命令 ip addr查看本机器在网卡上
#{mcast_src_ip 192.168.33.63} 为本节点ip地址
#{unicast_peer} 内容为其他master节点的ip
#{virtual_ipaddress} 为虚拟ip的地址

cat <<EOF > /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://192.168.33.66:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 61
    priority 80
    advert_int 1
    mcast_src_ip 192.168.33.63
    nopreempt
    authentication {
        auth_type PASS
        auth_pass sqP05dQgMSlzrxHj
    }
    unicast_peer {
        192.168.33.61
        192.168.33.62
    }
    virtual_ipaddress {
        192.168.33.66/24
    }
    track_script {
        CheckK8sMaster
    }

}
EOF
```

```bash
#master节点都启动 keepalived
systemctl restart keepalived
```

```bash
#使用命令ip addr 查看虚拟ip已经绑定到k8snode01上
ip addr
#看到如下信息证明虚拟ip已经绑定到master上
inet 192.168.33.66/24 scope global secondary eth1
       valid_lft forever preferred_lft forever
```




### 5.每台服务安装docker、kubeadm、kubelet、kubectl
```bash
#安装docker
#请查看https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.13.md 对应kubernetes 支持的版本
#The list of validated docker versions remain unchanged at 1.11.1, 1.12.1, 1.13.1, 17.03, 17.06, 17.09, 18.06 since Kubernetes 1.12.
#
# Set up repository
yum install -y yum-utils device-mapper-persistent-data lvm2

# Use Aliyun Docker
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#查看docker版本
yum list docker-ce --showduplicates
#安装指定版本的docker
yum install -y docker-ce-18.06.1.ce
#启动docker
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
systemctl status docker
```

```bash
#安装指定版本kubeadm、kubelet、kubectl
#查看可安装的版本
yum list kubeadm --showduplicates
# 进行安装，kubeadm、kubelet、kubectl 3个版本保持一样
yum install -y kubeadm-1.13.2-0 kubelet-1.13.2-0 kubectl-1.13.2-0
```

```bash
#添加命令补全
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
#启动kubelet
systemctl daemon-reload
systemctl enable kubelet
```



### 6.在k8snode01上初始化mster节点
```bash
#设置集群配置文件
#192.168.33.66 虚拟ip在哪个节点就在哪个节点初始化
#192.168.33.66 为虚拟ip 6443 为apiserver端口
cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "10.100.0.1"
  bindPort: 6443
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
etcd:
  local:
    imageRepository: "registry.cn-hangzhou.aliyuncs.com/google_containers"
    imageTag: "3.2.24"
    dataDir: "/var/lib/etcd"
    extraArgs:
      listen-client-urls: "https://192.168.33.61:2379,http://127.0.0.1:2379"
      advertise-client-urls: "https://192.168.33.61:2379"
      initial-advertise-peer-urls: "https://192.168.33.61:2380"
      initial-cluster: "k8snode01=https://192.168.33.61:2380"
      listen-peer-urls: "https://192.168.33.61:2380"
    serverCertSANs:
    -  "192.168.33.61"
    -  "192.168.33.66"
    -  "k8snode01"
    peerCertSANs:
    -  "192.168.33.61"
    -  "192.168.33.66"
    -  "k8snode01"
kubernetesVersion: "v1.13.2"
apiServer:
  certSANs:
  - "192.168.33.66"
controlPlaneEndpoint: "192.168.33.66:6443"
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.100.0.1/24"
  dnsDomain: "cluster.local"
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
clusterName: "k8s-cluster"
EOF

#集群初始化
kubeadm init --config=kubeadm-config.yaml
```

```bash
#看到如下信息，证明已经成功了
#保存以下信息，后续会用到
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.33.66:6443 --token xzffzn.2veb9rd3hzj5kfe8 --discovery-token-ca-cert-hash sha256:1e044b5cc57a86839bdfd0c71b42e37b7c6e9c5b786bbefe8ce9cabe3537a50a
```

```bash
#失败后的处理方式
kubeadm reset
rm -rf /etc/kubernetes/*.conf
rm -rf /etc/kubernetes/manifests/*.yaml
docker ps -a |awk '{print $1}' |xargs docker rm -f
systemctl  stop kubelet

ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
systemctl start kubelet
```

```bash
#根据提示执行以下语句，kubectl才有权限执行
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


```bash
#添加网络插件
#有多种插件可以选，任选一种就可以.更多其他网络插件请参考官网https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
#注意生产环境，请保存好yaml文件，方便后续使用kubectl delele -f 删除所有相关组件
#所以最好还是先 wget xxx 再 kubectl apply -f 本地文件

#flannel 方式
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml

```

```bash
#执行命令查看所有pod的状态，直到所有pod都启动完毕之后再进行
kubectl get pod -n kube-system -w
```



```bash
#如果添加maste节点，则需要把相关的证书传输到对应节点上
#在k8snode01上把初始化后的证书传输到其他master节点上
scp -r /etc/kubernetes/pki  k8snode02:/etc/kubernetes/
scp -r /etc/kubernetes/admin.conf  k8snode02:/etc/kubernetes/admin.conf
scp -r kubeadm-config.yaml  k8snode02:

scp -r /etc/kubernetes/pki  k8snode03:/etc/kubernetes/
scp -r /etc/kubernetes/admin.conf  k8snode03:/etc/kubernetes/admin.conf
scp -r kubeadm-config.yaml  k8snode03:

#登录到其他master节点上执行添加命令
route add default gw 192.168.33.1
#注意：比添加work节点多了 --experimental-control-plane
#修改config
sed -i "s/192.168.33.61/192.168.33.62/g" kubeadm-config.yaml
#加入集群
kubeadm join 192.168.33.66:6443 --token xzffzn.2veb9rd3hzj5kfe8 --discovery-token-ca-cert-hash sha256:1e044b5cc57a86839bdfd0c71b42e37b7c6e9c5b786bbefe8ce9cabe3537a50a --experimental-control-plane --apiserver-advertise-address 192.168.33.62
#删除默认路由
route del default gw 192.168.33.1

```


```bash
#添加node节点，在对应work节点上执行添加命令
kubeadm join 192.168.33.66:6443 --token 9dkawf.9evw3ngzmsxg1xmp --discovery-token-ca-cert-hash sha256:20f4e98e6231a3cd498a99a844d93c3f45f9991f16e245ce62ebe0d31d185875

```







## 参考
https://www.kubernetes.org.cn/3808.html
https://kubernetes.io/docs/setup/independent/install-kubeadm/
https://choerodon.io/zh/docs/installation-configuration/steps/kubernetes/
https://blog.csdn.net/nklinsirui/article/details/80610058
https://k8smeetup.github.io/docs/admin/kubeadm/#config-file
https://k8smeetup.github.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file
https://blog.csdn.net/networken/article/details/84571373
https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta1
https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
https://github.com/Lentil1016/kubeadm-ha/issues/34
https://kubernetes.io/docs/setup/independent/ha-topology/
## 让非root用户可以使用kubectl

#### 4.1. 创建用户并且设置用户可以使用sudo权限

```bash
#使用root用户新建用户
sudo groupadd k8sadm
sudo useradd k8sadm -g k8sadm -p 12345678
#修改/etc/sudoers 文件使新建的用户可以使用sudo权限
sudo chmod +w /etc/sudoers 
# 在 Allow root to run any commands anywhere 的下面 的 root 用户的下一行添加
#本人使用比较懒的方式
cat <<EOF >> /etc/sudoers
k8sadm     ALL=(ALL)       NOPASSWD:  ALL
EOF
#修改回原来的权限
sudo chmod -w /etc/sudoers
```





