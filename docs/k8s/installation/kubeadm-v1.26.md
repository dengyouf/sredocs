# Kubeadm 安装 Kubernetes v1.26.9(Containerd)

### 主机环境准备

使用Kubeadm快速部署Kubernetes集群，操作系统为Ubuntu 20.04.6 LTS，用到的各相关程序版本如下

- Kubernetes: v1.26.9
- Containerd: 1.7.27
- CNI: Flannel

#### 1. 主机名解析

```
echo "
192.168.1.31 k8s-master01 master01 k8s-api.linux.io
192.168.1.41 k8s-worker01 worker01
192.168.1.42 k8s-worker02 worker02
" |sudo tee -a /etc/hosts
```
#### 2. 系统基础软件环境

```
sudo apt-get update
sudo apt remove ufw lxd lxcfs lxc-common -y
sudo apt install -y bash-completion conntrack  ipset ipvsadm  jq libseccomp2  nfs-common  psmisc  rsync  socat
```
#### 3. 禁用系统swap

```
sudo sed  -ri  's@/.*swap.*@# &@' /etc/fstab 
sudo systemctl  mask $(sudo systemctl  --type swap|grep "loaded active"|awk '{print $1}')
sudo swapoff -a && sudo sysctl -w vm.swappiness=0
```

#### 4. 时间同步

```
sudo cp /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
sudo apt install  -y chrony

echo "
server ntp.aliyun.com iburst
stratumweight 0
driftfile /var/lib/chrony/drift
rtcsync
makestep 10 3
bindcmdaddress 127.0.0.1
bindcmdaddress ::1
keyfile /etc/chrony.keys
commandkey 1
generatecommandkey
logchange 0.5
logdir /var/log/chrony
"| sudo tee /etc/chrony/chrony.conf

sudo systemctl  restart chrony
chronyc  sources
```

#### 5.  优化日志

优化设置 journal 日志相关，避免日志重复搜集，浪费系统资源

```
sudo mkdir -pv /etc/systemd/journald.conf.d

echo "[Journal]
# 持久化保存到磁盘
Storage=persistent
# 最大占用空间 2G
SystemMaxUse=2G
# 单日志文件最大 200M
SystemMaxFileSize=200M
# 日志保存时间 2 周
MaxRetentionSec=2week
# 禁止转发
ForwardToSyslog=no
ForwardToWall=no"|sudo tee  /etc/systemd/journald.conf.d/95-k8s-journald.conf

sudo systemctl restart systemd-journald 
```

#### 6. 自动加载模块

启用systemd自动加载模块服务

```
echo "
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
" |sudo tee /etc/modules-load.d/10-k8s-modules.conf
sudo systemctl enable systemd-modules-load  --now
```

#### 7. 内核参数优化

```
echo "
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-arptables = 1
net.ipv4.tcp_tw_reuse = 0
net.core.somaxconn = 32768
net.netfilter.nf_conntrack_max=1000000
vm.swappiness = 0
vm.max_map_count=655360
fs.file-max=6553600
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
" | sudo tee /etc/sysctl.d/95-k8s-sysctl.conf

sudo sysctl -p /etc/sysctl.d/95-k8s-sysctl.conf
```

#### 8. 设置系统 ulimits

```
sudo mkdir /etc/systemd/system.conf.d

echo "
[Manager]
DefaultLimitCORE=infinity
DefaultLimitNOFILE=100000
DefaultLimitNPROC=100000
"| sudo tee /etc/systemd/system.conf.d/30-k8s-ulimits.conf

```

- 把SCTP列入内核模块黑名单

```
echo "
install sctp /bin/true
"|sudo tee /etc/modprobe.d/sctp.conf
```

- 重启机器

```
sudo reboot
```

### 安装容器运行时

```
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

# 安装必要的一些系统工具
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# 信任 Docker 的 GPG 公钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 写入软件源信息
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 containerd
sudo apt update
sudo apt install -y containerd.io

# 使用systemd作为cgroup
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's@SystemdCgroup \= false@SystemdCgroup \= true@g' /etc/containerd/config.toml
sudo sed -i 's@registry.k8s.io/pause:3.8@registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9@g' /etc/containerd/config.toml

# 启动服务并设置为自启动
sudo systemctl restart containerd
sudo systemctl enable containerd
```

```
sudo mkdir /etc/systemd/system/containerd.service.d
echo "
[Service]
Environment="HTTP_PROXY=http://192.168.1.102:7890"  
Environment="HTTPS_PROXY=http://192.168.1.102:7890" 
Environment="NO_PROXY=localhost,127.0.0.0/8,10.244.0.0/16,172.16.0.0/12,10.96.0.0/12,.svc,.cluster.local,.linux.io"
"| sudo tee /etc/systemd/system/containerd.service.d/http-proxy.conf

sudo systemctl daemon-reload
sudo systemctl restart containerd.service
```

###  安装kubeadm

```
# 设置阿里kubernetres源
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/kubernetes-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/etc/apt/trusted.gpg.d/kubernetes-archive-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 安装kubeadm
sudo apt update
sudo apt install -y kubeadm=1.26.9-00 kubelet=1.26.9-00 kubectl=1.26.9-00
sudo apt-mark hold kubelet kubeadm kubectl
```

### 初始化集群

```
sudo kubeadm config images pull --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --kubernetes-version=1.26.9
```

#### 1. 控制平面初始化

```
sudo kubeadm init --kubernetes-version=v1.26.9 \
    --control-plane-endpoint=k8s-api.linux.io \
    --apiserver-advertise-address=0.0.0.0 \
    --pod-network-cidr=10.244.0.0/16   \
    --service-cidr=10.96.0.0/12 \
    --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers \
    --token-ttl=0 \
    --upload-certs \
    --ignore-preflight-errors=Swap

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8s-api.linux.io:6443 --token 6dg4xz.zxima0m81fw7ikfg \
	--discovery-token-ca-cert-hash sha256:60108201f09d1acc5ad9f010baadb4bd4240fd9e7a44d06c18ce6fe6b8b4aa87 \
	--control-plane --certificate-key 3b63422672a10b4cd313bd9a32516f1cac4a240f6939a74144f5d0c666a7a1dc

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-api.linux.io:6443 --token 6dg4xz.zxima0m81fw7ikfg \
	--discovery-token-ca-cert-hash sha256:60108201f09d1acc5ad9f010baadb4bd4240fd9e7a44d06c18ce6fe6b8b4aa87
```

#### 2. 配置Kubectl客户端

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 3. 部署Flannel

```
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
kubectl apply -f kube-flannel.yml
```

#### 4. 添加Worker节点

```
sudo kubeadm join k8s-api.linux.io:6443 --token 6dg4xz.zxima0m81fw7ikfg \
	--discovery-token-ca-cert-hash sha256:60108201f09d1acc5ad9f010baadb4bd4240fd9e7a44d06c18ce6fe6b8b4aa87
```

### 修改网络模式IPVS

```
~$ kubectl  edit cm/kube-proxy -n kube-system -o yaml
  ...
  metricsBindAddress: ""
  mode: "ipvs" #
  nodePortAddresses: null
  oomScoreAdj: null
  ...

~$ kubectl  delete pod -n kube-system -l k8s-app=kube-proxy
~$ sudo ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 192.168.1.31:6443            Masq    1      0          0
TCP  10.96.0.10:53 rr
  -> 10.244.1.2:53                Masq    1      0          0
  -> 10.244.1.3:53                Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 10.244.1.2:9153              Masq    1      0          0
  -> 10.244.1.3:9153              Masq    1      0          0
UDP  10.96.0.10:53 rr
  -> 10.244.1.2:53                Masq    1      0          0
  -> 10.244.1.3:53                Masq    1      0          0
```

### 验证集群

- 创建 Deployment 和 Service 资源

```
~$ kubectl  get svc/myapp
NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
myapp   NodePort   10.97.153.71   <none>        80:30488/TCP   9s
~$ kubectl  get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
myapp-6454786f57-cx7wm   1/1     Running   0          23s   10.244.2.2   k8s-worker02   <none>           <none>
myapp-6454786f57-fs8fz   1/1     Running   0          23s   10.244.1.4   k8s-worker01   <none>           <none>
myapp-6454786f57-kl64l   1/1     Running   0          23s   10.244.2.3   k8s-worker02   <none>           <none>
```

```
for i in `seq 5`;do curl http://10.97.153.71/hostname.html;done
myapp-6454786f57-kl64l
myapp-6454786f57-cx7wm
myapp-6454786f57-fs8fz
myapp-6454786f57-kl64l
myapp-6454786f57-cx7wm
```

> 注意：Service IP 和 NodePort 仅是网络规则，所以不支持ping， 但是支持telnet

- 验证网络

service 的默认域名为 SVC_NAME.NAMESPACE.svc.cluster.local 
```
~$ kubectl  run busybox --image=busybox:1.28 -- sleep 3600

~$ kubectl  exec -it busybox -- nslookup myapp.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      myapp.default.svc.cluster.local
Address 1: 10.97.153.71 myapp.default.svc.cluster.local
```