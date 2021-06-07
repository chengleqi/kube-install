```sh
# 重置hostname
hostnamectl set-hostname your-new-host-name

# 设置主机名解析
vi /etc/hosts
192.168.174.100 master
192.168.174.101 node1
192.168.174.102 node2
```

```sh
# 停止并禁用firewalld
systemctl stop firewalld
systemctl disable firewalld
```

```sh
# 关闭selinux
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

```sh
# 关闭swap
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab
```

```sh
# 编辑文件/etc/sysctl.d/kubernetes.conf文件
vi /etc/sysctl.d/kubernetes.conf
# 添加如下配置
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
# 重新加载配置
sysctl -p
```

```sh
# 加载网桥过滤模块
modprobe br_netfilter
# 查看网桥过滤模块是否加载成功
lsmod | grep br_netfilter
```

```sh
# 更新yum源为清华源
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
	-e 's|^#baseurl=http://mirror.centos.org|baseurl=https://mirrors.tuna.tsinghua.edu.cn|g' \
	-i.bak \
	/etc/yum.repos.d/CentOS-*.repo
	
# 更新缓存
yum makecache

# 配置ipvs功能
yum install ipset ipvsadm -y

cat <<EOF>  /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

# 为脚本文件添加执行权限
chmod +x /etc/sysconfig/modules/ipvs.modules

# 执行脚本文件 
/bin/bash /etc/sysconfig/modules/ipvs.modules

# 查看对应的模块是否加载成功
lsmod | grep -e ip_vs -e nf_conntrack
```

```sh
# 安装docker
dnf remove podman
yum erase podman buildah
yum install docker-ce docker-ce-cli containerd.io
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://dzeem0j4.mirror.aliyuncs.com"]
}
EOF
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
docker version
```

```sh
# 安装kubernetes组件
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum makecache
yum install -y kubelet-1.21.1-0 kubeadm-1.21.1-0 kubectl-1.21.1-0
```

```sh
# 配置kubelet的cgroup
vi /etc/sysconfig/kubelet

KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"

systemctl enable kubelet
```

```sh
# 查看组件版本
kubeadm config images list

k8s.gcr.io/kube-apiserver:v1.21.1
k8s.gcr.io/kube-controller-manager:v1.21.1
k8s.gcr.io/kube-scheduler:v1.21.1
k8s.gcr.io/kube-proxy:v1.21.1
k8s.gcr.io/pause:3.4.1
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns/coredns:v1.8.0
```

```sh
# 安装组件
images=(
	kube-apiserver:v1.21.1
	kube-controller-manager:v1.21.1
	kube-scheduler:v1.21.1
	kube-proxy:v1.21.1
	pause:3.4.1
	etcd:3.4.13-0
	coredns:1.8.0
)
for imageName in ${images[@]} ; do
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
	docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done

docker tag k8s.gcr.io/coredns:1.8.0 k8s.gcr.io/coredns/coredns:v1.8.0
docker rmi k8s.gcr.io/coredns:1.8.0
```

```sh
# 创建集群
kubeadm init \
--kubernetes-version=1.21.1 \
--pod-network-cidr=10.244.0.0/16 \
--service-cidr=10.96.0.0/12 \
--apiserver-advertise-address=192.168.174.100

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.174.100:6443 --token 28vcgo.i27c4gyzu4i2as6z \
	--discovery-token-ca-cert-hash sha256:71836ee36e1e93bd6ab5b790fcd225b9776b3cd77e079187180cfeb124b8e615
```

```sh
# 安装网络插件
kubectl apply -f kube-flannel.yml
# 其中kube-flannel.yml中两处quay.io都更改为中科大镜像quay.mirrors.ustc.edu.cn
```

