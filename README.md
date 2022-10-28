# 服务器准备
1. 一台作为Master的服务器（2vCPU, 4G, Ubuntu 20.04）
2. 一台作为Worker的服务器（2vCPU, 4G, Ubuntu 20.04）



# 安装Kubeadm (两台服务器都需要做以下操作)
    由于以下安装Kubeadm的步骤，两台服务器都需要做，其实可以先只准备一台服务器，将以下步骤做完后，再克隆出另外一台服务器，区别只是需要将hostname改为不一样

### 安装docker, 修改配置
```bash
apt update       # 记得替换为阿里里apt源
vi /etc/hostname # 修改hostname, 这将集群中node的名字，比如一台修改为master，另一台修改为worker


apt install -y docker.io   # 安装docker
service docker start       # 启动docker，docker version可验证docker是否安装好
usermod -aG docker ${USER} # 当前用户加入docker组，若直接用root用户，可忽略此步骤

# cgroup驱动程序改成systemd
cat <<EOF | sudo tee /etc/docker/daemon.json
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {"max-size": "100m"},
    "storage-driver": "overlay2"
}
EOF

# 设置docker开机启动，并重启docker
systemctl enable docker
systemctl daemon-reload
systemctl restart docker


# 为了让k8s能够检查、转发网络流量，需要修改iptables的配置，启用br_netfilter模块
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1 # better than modify /etc/sysctl.conf
EOF

sudo sysctl --system


# 修改/etc/fstab，关闭Linux的swap分区，提升k8s的性能
swapoff -a
sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
```

### 安装kubeadm、kubelet、kubectl
```bash
apt install -y apt-transport-https ca-certificates curl

# 下载k8s源
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.listdeb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt update

# 记住我们这里使用1.23.3的版本，之后在使用kubeadm init时还会用到
apt install -y kubeadm=1.23.3-00 kubelet=1.23.3-00 kubectl=1.23.3-00 
kubeadm version          # 验证kubeadm
kubectl version --client # 验证kubectl
```



# 安装Master节点（仅在Mater服务器上做）
```bash
# --apiserver-advertise-address为master服务器的ip。比如在阿里云ECS中，为ECS的私网ip
# --kubernetes-version需要为之前安装kubeadm是用的版本，这里用的是v1.23.3
# --service-cidr和--pod-network-cidr不要与局域网子网（vpc子网）冲突
kubeadm init \
--apiserver-advertise-address=x.x.x.x \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.23.3 \
--service-cidr=10.80.0.0/16 \
--pod-network-cidr=192.180.0.0/16

# kubeadm init成功后，按照提示执行以下命令
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 将提示中类似如下的命令保存下来，之后创建worker节点时需要用到
kubeadm join x.x.x.x:6443 --token tv9mkx.tw7it9vphe158e74 \ --discovery-token-ca-cert-hash sha256:e8721b8630d5b562e23c010c70559a6d3084f629abad6a2920e87855f8fb96f3

# 安装Flannel网络插件，也可以直接使用仓库中我提前下载下来的kube-flannel.yml
wget https://github.com/flannel-io/flannel/raw/master/Documentation/kube-flannel.yml

# 把kube-flannel.yml文件中的'net-conf.json'字段的'Network'项，改成刚刚--pod-network-cidr中设置的pod地址段, 如下
net-conf.json: |
{
    "Network": "192.180.0.0/16",
    "Backend": {
    "Type": "vxlan"
    }
}

# 用docker pull尝试kube-flannel.yml中的用到的镜像能否下载下来，如果不能则需要借助dockerhub或aliyun registry, 并结合docker tag想办法将这两镜像pull下来
docker pull docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:x.x.x
docker pull docker.io/rancher/mirrored-flannelcni-flannel:x.x.x

# 等一段时间后，查看node状态为Ready则说明成功
kubectl apply -f kube-flannel.yml
```



# 安装Worker节点（仅在Worker服务器上做）
```bash
# 将之前生成的kubeadm join命令在worker服务器上执行，如下
kubeadm join x.x.x.x:6443 --token tv9mkx.tw7it9vphe158e74 \ --discovery-token-ca-cert-hash sha256:e8721b8630d5b562e23c010c70559a6d3084f629abad6a2920e87855f8fb96f3

# 等一段时间后，查看node状态为Ready则说明成功
```


# 在非node节点使用kubectl
按照上述步骤，只能在master节点使用kubectl来访问集群资源
若想在其它服务器（比如和集群无关的跳板机或控制台服务器）上也能通过kubectl访问k8s资源:
1. 服务器，和k8s节点服务器需要在同一局域网（否则得使用使用公网SLB等计入方式）、
2. 将master服务器上的kubectl二进制程序，拷贝到服务器上；或直接在服务器上，安装相同版本的kubectl
3. 将master服务器上的~/.kube/config文件，拷贝到服务器的~/.kube/目录下


# kubectl自动补全
```bash
apt install -y bash-completion
source /usr/share/bash-completion/bash_completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
```



# 安装kubens, 快速切换namespace
```bash
git clone https://github.com/ahmetb/kubectx
ln -s kubectx/kubens /usr/local/bin/kubens
chmod u+x /usr/local/bin/kubens
```
