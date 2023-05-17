[TOC]

## kubernetes v1.25.9部署

####  环境准备：

| 主机名 |    IP地址     |    操作系统    |            硬盘            |
| :----: | :-----------: | :------------: | :------------------------: |
| master | 192.168.88.30 | CentOS7.9  5.4 | CPU:2核  内存:4G  硬盘:50G |
| node1  | 192.168.88.31 | CentOS7.9  5.4 | CPU:2核  内存:4G  硬盘:50G |
| node2  | 192.168.88.32 | CentOS7.9  5.4 | CPU:2核  内存:4G  硬盘:50G |

### 操作系统环境准备：

##### 禁用SELinux

```
# 所有主机都做
sed -i 's,SELINUX=enforcing,SELINUX=disabled,g' /etc/selinux/config
或
vim /etc/selinux/config   #修改enforcing为disabled

setenforce 0     # 临时修改selinux状态
getenforce       # 查看当前selinux状态
```

##### 关闭firewalld

```
# 所有主机都做
# 关闭防火墙或卸载
systemctl stop firewalld       # 停止firewalld
systemctl disable firewalld    # 禁止firewalld开机自启
systemctl status firewalld     # 查看firewalld当前状态
或
yum remove -y firewalld        # 卸载firewalld
```

##### 禁用swap交换分区

```
# 所有主机都做
swapoff -a   # 禁用swap交换分区
vim /etc/fstab    # 注释掉交换分区挂载命令，永久禁止
free -h      # 查看交换分区禁用情况
```

##### 设置本地域名解析

```
# 用同步输入模式将三台主机都设置解析
cat >> /etc/hosts <<EOF
192.168.88.30  master
192.168.88.31  node1
192.168.88.32  node2
EOF
# 或用scp等方式传输
scp /etc/hosts  192.168.88.31:/etc/    # 覆盖原本/etc/hosts
^31^32     # 将以上指令中的31替换为32再执行一遍    
```

##### 配置免密登录

```
# 生成公私钥
ssh-keygen   # 全部回车

# 发送公钥到被控节点
for i in ｛31..32｝
do
ssh-copy-id 192.168.88.$i
done

# yes
```

##### 时间同步

```
# chrony默认都会安装；所有主机都做
vim /etc/chrony.conf
server ntp1.aliyun.com iburst
# 添加ntp1.aliyun.com 这是阿里云开放的NTP同步地址
```

##### 修改内核参数

```
# 所有主机
cat > /etc/sysctl.d/k8s/conf <<EOF   # 将桥接的IPV4流量传递到iptables的链里
vm.swappiness = 0
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system 或 sysctl -p     # 重新加载系统参数
```

##### 升级内核

```
# 查看当前内核版本
uname -ar
Linux master 3.10.0-1160.el7.x86_64 #1 SMP Mon Oct 19 16:18:59 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux

# 导入公钥
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

# 下载并安装elrepo仓库
yum install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm


# 下载相关包，一般默认已安装
yum install yum-plugin-fastestmirror

# 查看elrepo源里有什么版本的内核，lt为长期稳定版本，ml为常更新版本
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available

# 安装长期稳定版本，稳定可靠
yum --enablerepo=elrepo-kernel install kernel-lt	

# 查看内核是否载入到grub2
sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg

# 修改默认版本内核
vim /etc/default/grub
GRUB_DEFAULT=0

# 生成 grub 配置文件
grub2-mkconfig -o /boot/grub2/grub.cfg

# 重启系统
init6  或  reboot

# 查看当前版本
uname -ar
Linux node1 5.4.242-1.el7.elrepo.x86_64 #1 SMP Tue Apr 25 09:46:07 EDT 2023 x86_64 x86_64 x86_64 GNU/Linux
```

##### 安装基础软件包

```
yum -y install yum-utils device-mapper-persistent-data  wget net-tools nfs-utils lrzsz gcc make openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim  autoconf automake zlib-devel python-devel  openssh-server socat ipvsadm conntrack  telnet ipvsadm openssh-clients bash-completion 
```

### 安装containerd

```
# 在所有节点操作，containerd是作为docker的组件在安装docker的时候会被安装上的，无需再额外执行yum install -y containerd，如遇到相同问题可忽略报错

yum -y install yum-utils   # 提供yum-config-manger命令
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum -y install docker-ce   # 安装最新版docker

# 启动docker
systemctl start docker
systemctl enable docker
```

##### 配置docker镜像加速

```
cat > /etc/docker/daemon.json <<EOF
{
"registry-mirrors": ["https://cy3j0usn.mirror.aliyuncs.com",“https://registry.grc.io”],
"exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

# 重新加载配置
systemctl daemon-reload

# 重启docker
systemctl restart docker
```

##### 修改containerd配置文件（所有节点）

```
# 创建配置文件目录
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml   # 生成config.toml配置文件

# 修改配置文件
vim /etc/containerd/config.toml

# 修改cgroup驱动为systemd，把SystemdCgroup = false 改为 true
SystemdCgroup = true

# 把sandbox_image = "k8s.gcr.io/pause:3.6" 修改为以下内容
sandbox_image="registry.aliyuncs.com/google_containers/pause:3.7"

# 把 config_path = "" 修改为以下内容
config_path = "/etc/containerd/certs.d"

# containerd设置开机自启
systemctl enable containerd --now
```

##### 配置containerd镜像加速（所有节点）

```
# 创建目录树
mkdir -p /etc/containerd/certs.d/docker.io/

# 编写镜像加速配置文件
cat >/etc/containerd/certs.d/docker.io/hosts.toml<<EOF
[host."https://cy3j0usn.mirror.aliyuncs.com","https://registry.gcr.io"]
capbilities = ["pull"]
EOF

# 重启containerd
systemctl restart containerd
```

##### 配置Runtime为containerd（所有节点）

```
# 编写/etc/crictl.yaml文件
cat > /etc/crictl.yaml << EOF
runtime-endpoint:unix:///run/containerd/containerd.sock
image-endpoint:unix:///run/containerd/containerd.sock
timeout:10
debug:false
EOF

# 重启containerd
systemctl restart containerd
```

##### 配置阿里云Yum源（所有节点）

```
cd /etc/yum.repos.d/

# 下载阿里云yum仓库
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# 添加docker源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 修改yum配置
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
EOF

# 下载v1.25.9相关工具
yum install -y kubelet-1.25.9 kubeadm-1.25.9 kubectl-1.25.9

# 设置kubelet开机自启，kubelet状态执行kubeadm init后才会running
systemctl enable kubelet
```

### 初始化kubernetes集群

##### 设置Runtime（所有节点）

```
# 建立containerd通信连接
crictl config runtime-endpoint /run/containerd/containerd.sock

# 导出默认配置文件
kubeadm config print init-defaults > kubeadm.yaml
```

##### 修改初始化配置文件

```
# 修改kubeadm.yaml
vim kubeadm.yaml

# 修改添加以下带有注释的行
apiVersion: kubeadm.k8s.io/v1beta3
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
  advertiseAddress: 192.168.88.30              # 修改为masterIP地址
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: master                            # 修改为master主机名
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers  # 修改镜像仓库
kind: ClusterConfiguration
kubernetesVersion: 1.25.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16           	 # 新增一行指定pod网段
scheduler: {}

# 在末行添加以下字段
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

##### 初始化集群

```
# 执行初始化集群命令
kubeadm init --config=kubeadm.yaml --ignore-preflight-errors=SystemVerification

# 成功时会生成你的token
例：kuadm join 192.168.88.30 --token abcdef.0123456789abcdef ......

# 初始化成功后的根据提示执行以下命令
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

##### node节点加入集群

```
# 在master执行kubeadm token create --print-join-command
# 将获取结果尾部添加 --ignore-preflight-errors=SystemVerification复制到node节点执行
例：kubeadm join 192.168.88.30:6443 --token ces4y0.ot8a5xq0d1uqbnlk --discovery-token-ca-cert-hash sha256:88f1fb41f7ee073362ce831be9baa68bd576b2953f2c01dd6068f87031ff94ae --ignore-preflight-errors=SystemVerification

# 查看节点是否加入集群
kubectl get nodes
```

##### 安装cni网络插件

```
# 下载资源对象文件
wget -O https://docs.projectcalico.org/manifests/calico.yaml

# 导入
kubectl apply -f calico.yaml

# 等待一段时间后查看节点状态是否为Ready
kubectl get nodes
```

