# Kubeadm 安装 Kubernetes v1.28.10(Docker)

### 主机环境准备

使用Kubeadm快速部署Kubernetes集群，操作系统为Ubuntu 20.04.6 LTS，用到的各相关程序版本如下

Kubernetes: v1.28.6
Docker: 24.0.8
Cri-dockerd: v0.3.8

#### 1. 主机名解析

```shell
cat >> /etc/hosts << 'EOF'
192.168.1.201 k8s-master01
192.168.1.211 k8s-worker01
192.168.1.212 k8s-worker02
192.168.1.213 k8s-worker03
# k8s-vip 预留后期扩展高可用集群
192.168.1.201 k8s-vip
EOF
```


#### 2. 关闭防火墙和selinux(Ubuntu不用管SELinux)

```shell
systemctl disable --now ufw
```

#### 3. 关闭swap分区

```shell
sed -i 's/.*swap.*/#&/' /etc/fstab
swapoff -a && sysctl -w vm.swappiness=0

systemctl  mask swap.target
```

#### 4. 时区跟时间同步

```shell
timedatectl  set-timezone Asia/Shanghai

apt install -y chrony
cat  > /etc/chrony/chrony.conf << 'EOF'
pool ntp.aliyun.com       iburst maxsources 4
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
logdir /var/log/chrony
maxupdateskew 100.0
rtcsync
makestep 1 3
EOF

systemctl restart chrony.service && systemctl  enable chrony.service

chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 203.107.6.88                  2   6    37    14   +450us[ +336us] +/-   30ms
```

#### 5. 加载IPVS模块

```shell
apt install ipset ipvsadm -y

tee  /etc/modules-load.d/k8s.conf << 'EOF'
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
nf_nat
overlay
vxlan
EOF
systemctl restart systemd-modules-load
lsmod | grep -E 'br_netfilter|ip_vs|nf_conntrack|overlay|vxlan'
```

#### 6. 内核参数优化

```shell
tee  /etc/sysctl.d/k8s.conf << 'EOF'
# 开启IPv4转发
net.ipv4.ip_forward = 1
# 允许桥接的流量进入iptables/netfilter
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
# 优化连接跟踪表大小，防止大规模连接爆掉
net.netfilter.nf_conntrack_max = 2310720
# TCP优化（缩短 TIME_WAIT，快速回收连接）
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
# 内存相关优化（防止OOM）
vm.swappiness = 0
vm.overcommit_memory = 1
vm.panic_on_oom = 0
# 文件句柄限制
fs.file-max = 52706963
# 网络层面优化
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_syn_backlog = 16384
EOF
sysctl --system
```

```shell
tee -a /etc/security/limits.conf <<EOF
* soft nofile 1048576
* hard nofile 1048576
* soft nproc 1048576
* hard nproc 1048576
* soft memlock unlimited
* hard memlock unlimited
EOF
```

### 安装容器运行时

#### 1. 安装docker

```shell
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# step 2: 信任 Docker 的 GPG 公钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Step 3: 写入软件源信息
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update

# Step 4: 安装Docker
apt-get update
apt-get install docker-ce=5:24.0.6-1~ubuntu.20.04~focal
```
```shell
tee  /etc/docker/daemon.json << 'EOF'
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": [
         "https://o4uba187.mirror.aliyuncs.com",
         "https://docker.1ms.run",
         "https://docker.1panel.live"
    ]
}
EOF
systemctl daemon-reload
apt-cache madison docker-ce
systemctl restart docker && systemctl enable docker
```

#### 2. 安装cri-dockerd

```shell
#curl -LO https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.10/cri-dockerd_0.3.10.3-0.ubuntu-focal_amd64.deb
curl -LO https://ghfast.top/https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.10/cri-dockerd_0.3.10.3-0.ubuntu-focal_amd64.deb
apt install  -y ./cri-dockerd_0.3.10.3-0.ubuntu-focal_amd64.deb
```
```shell
cp /lib/systemd/system/cri-docker.service{,.bak}
sed -i 's@ExecStart=/usr/bin/cri-dockerd@ExecStart=/usr/bin/cri-dockerd --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9 @g' /usr/lib/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl restart cri-docker && systemctl  enable  cri-docker
```

### 安装Kubernetes集群

#### 1. 安装软件包

```shell
apt-get update && apt-get install -y apt-transport-https
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
# apt-cache madison kubeadm
apt install kubeadm=1.28.10-1.1 kubelet=1.28.10-1.1 kubectl=1.28.10-1.1 -y
```

#### 2. 配置kubelet

```shell
sed -i 's@KUBELET_EXTRA_ARGS=@KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"@' /etc/sysconfig/kubelet
systemctl enable kubelet
```

#### 3. 初始化集群

- master节点初始化

```shell

# kubeadm config images pull --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --kubernetes-version=1.28.10    --cri-socket=unix:///var/run/cri-dockerd.sock
kubeadm init \
  --kubernetes-version=1.28.10 \
  --control-plane-endpoint="k8s-vip" \
  --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --token-ttl=0 \
  --upload-certs \
  --cri-socket=unix:///var/run/cri-dockerd.sock 

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

  kubeadm join k8s-vip:6443 --token r4818h.uij0z3hsaxdnhxbc \
	--discovery-token-ca-cert-hash sha256:1db36f3ec9ef152e106bbf94e0ba9afc8235ce4f1b6a1cfddd2016c87023d320 \
	--control-plane --certificate-key f7e2789c31298a2b4ac17e2079c6125a0395e701ef0cd3a6e82c959be3c49ae0

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-vip:6443 --token r4818h.uij0z3hsaxdnhxbc \
	--discovery-token-ca-cert-hash sha256:1db36f3ec9ef152e106bbf94e0ba9afc8235ce4f1b6a1cfddd2016c87023d320
```
- 配置kubectl客户端

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl  get nodes
NAME           STATUS     ROLES           AGE   VERSION
k8s-master01   NotReady   control-plane   55s   v1.28.10
```

- 加入worker节点

```shell
kubeadm join k8s-vip:6443 --token r4818h.uij0z3hsaxdnhxbc \
	--discovery-token-ca-cert-hash sha256:1db36f3ec9ef152e106bbf94e0ba9afc8235ce4f1b6a1cfddd2016c87023d320 \
	--cri-socket=unix:///var/run/cri-dockerd.sock
	
kubectl  get nodes
NAME           STATUS     ROLES           AGE     VERSION
k8s-master01   NotReady   control-plane   4m22s   v1.28.10
k8s-worker01   NotReady   <none>          34s     v1.28.10
k8s-worker02   NotReady   <none>          11s     v1.28.10
k8s-worker03   NotReady   <none>          7s      v1.28.10
```

#### 4. 部署网络插件

!!! warn "版本对应关系"
    需注意版本对应关系：https://docs.tigera.io/calico/3.27/getting-started/kubernetes/requirements

```shell
# curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.5/manifests/calico.yaml -O
curl -O https://ghfast.top/https://raw.githubusercontent.com/projectcalico/calico/v3.27.5/manifests/calico.yaml
sed -i 's@# - name: CALICO_IPV4POOL_CIDR@- name: CALICO_IPV4POOL_CIDR@g' calico.yaml
sed -i 's@#   value: "192.168.0.0/16"@  value: "10.244.0.0/16"@g' calico.yaml
kubectl  apply -f calico.yaml
```

#### 5. 修改网络模型为IPVS

```shell
kubectl  edit cm/kube-proxy -n kube-system -o yaml
  ...
  metricsBindAddress: ""
  mode: "ipvs" #
  nodePortAddresses: null
  oomScoreAdj: null
  ...
  
kubectl -n kube-system rollout restart daemonset kube-proxy
 ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 192.168.1.201:6443           Masq    1      1          0
TCP  10.96.0.10:53 rr
  -> 10.244.39.193:53             Masq    1      0          0
  -> 10.244.39.194:53             Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 10.244.39.193:9153           Masq    1      0          0
  -> 10.244.39.194:9153           Masq    1      0          0
UDP  10.96.0.10:53 rr
  -> 10.244.39.193:53             Masq    1      0          0
  -> 10.244.39.194:53             Masq    1      0          0
```

### 验证集群

#### 1. 验证网络

```shell
kubectl  create deployment myapp --image=ikubernetes/myapp:v1 --replicas=5
kubectl  get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
myapp-5d9c4b4647-27m7j   1/1     Running   0          9m21s   10.244.69.195   k8s-worker02   <none>           <none>
myapp-5d9c4b4647-2p6fs   1/1     Running   0          9m21s   10.244.79.70    k8s-worker01   <none>           <none>
myapp-5d9c4b4647-4dr5g   1/1     Running   0          3s      10.244.69.197   k8s-worker02   <none>           <none>
myapp-5d9c4b4647-p24mm   1/1     Running   0          9m21s   10.244.79.68    k8s-worker01   <none>           <none>
myapp-5d9c4b4647-vlsft   1/1     Running   0          28s     10.244.39.201   k8s-worker03   <none>           <none>
```
```shell
kubectl expose deployment myapp --port=80 --target-port=80 --type="NodePort" 
kubectl  get svc/myapp
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
myapp   NodePort   10.99.179.210   <none>        80:32178/TCP   20s
```
```shell
for i in `seq 5`;do curl 10.99.179.210/hostname.html;done
myapp-5d9c4b4647-2p6fs
myapp-5d9c4b4647-p24mm
myapp-5d9c4b4647-4dr5g
myapp-5d9c4b4647-27m7j
myapp-5d9c4b4647-vlsft
```

#### 2. 验证dns

```shell
apt install -y dnsutils

~# kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   44m
root@k8s-master01:~# dig -t A myapp.default.svc.cluster.local @10.96.0.10

; <<>> DiG 9.18.30-0ubuntu0.20.04.2-Ubuntu <<>> -t A myapp.default.svc.cluster.local @10.96.0.10
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45538
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: fd1fdbe4c0177389 (echoed)
;; QUESTION SECTION:
;myapp.default.svc.cluster.local. IN	A

;; ANSWER SECTION:
myapp.default.svc.cluster.local. 30 IN	A	10.99.179.210

;; Query time: 0 msec
;; SERVER: 10.96.0.10#53(10.96.0.10) (UDP)
;; WHEN: Sat Apr 26 18:44:31 CST 2025
;; MSG SIZE  rcvd: 119
```

### Add-On插件

#### 1. kuboard

使用 hostPath 提供持久化存储，将 kuboard 所依赖的 Etcd 部署到 Master 节点，并将 etcd 的数据目录映射到 Master 节点的本地目录；推

```shell
kubectl apply -f https://addons.kuboard.cn/kuboard/kuboard-v3-swr.yaml
```

#### 2. metrics-server


通过 kuboard 安装 metrics-server

```shell
kubectl  top nodes
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master01   133m         6%     1904Mi          24%
k8s-worker01   112m         5%     2325Mi          29%
k8s-worker02   63m          3%     1862Mi          23%
k8s-worker03   80m          4%     1965Mi          25%
kubectl  top pods
NAME                     CPU(cores)   MEMORY(bytes)
myapp-5d9c4b4647-27m7j   0m           2Mi
myapp-5d9c4b4647-2p6fs   0m           2Mi
myapp-5d9c4b4647-4dr5g   0m           2Mi
myapp-5d9c4b4647-p24mm   0m           2Mi
myapp-5d9c4b4647-vlsft   0m           2Mi
root@k8s-master01:~#
```

### 卸载集群

- 卸载整个集群

先拆除各个工作节点，在拆除控制平面

```shell
kubeadm reset  --cri-socket=unix:///var/run/cri-dockerd.sock
rm -rf /etc/kubernetes /var/lib/kubelet/ /var/lib/cni/ /etc/cni/net.d/ /var/lib/etcd/
```

- 拆除单个工作节点

```shell
# 1. 禁止调度
kubectl cordon k8s-worker03
# 2. 排空节点
kubectl drain k8s-worker03
# 3. 删除节点
kubectl delete node  k8s-worker03
# 4. 执行reset跟后续的清理工作
kubeadm reset  --cri-socket=unix:///var/run/cri-dockerd.sock
rm -rf /etc/kubernetes /var/lib/kubelet/ /var/lib/cni/ /etc/cni/net.d/ /var/lib/etcd/
```