---
title: "Kubernetes Single Master Cluster Install (CRI-O, Debian 12, >v1.24)"
date: 2024-06-09T02:40:35+08:00 
lastmod: 2024-06-09T02:40:35+08:00 
categories: 
- Kubernetes
tags: 
- Kubernetes
- crio
- Debian
summary: "安裝 K8s v1.24之後版本 (單個master node, crio, Debian 12)"
description: "安裝 K8s v1.24之後版本 (單個master node, crio, Debian 12)" 
weight: 
slug: ""
draft: false 
hidemeta: false
cover:
    image: "" 
    caption: ""
    alt: ""
    relative: false
---
# 說明
* 此份為建立 k8s 的第一步 - K8S Master & Worker 調適與安裝。
* 此為Kubernetes Cluster安裝文件，只有單個master，適用於Debian12，所有步驟worker, master都要執行，除了 **(Optional)** 跟 **(master操作即可)**。
* 安裝前先參考 [TODO]
* 若想要安裝多個master的HA架構，請參考 [TODO] 
* 若需要CentOS安裝教學，請參考 [TODO]
* [主要參考教學文章](https://insujang.github.io/2019-11-21/installing-kubernetes-and-crio-in-debian/)


# 操作步驟
## 從底層調整OS各項設定
1. 關閉SWAP記憶體與系統參數 

```bash
su - //用單純su不行
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab

// 檢查是否成功
// 成功關閉swap
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       270Mi       3.3Gi       4.0Mi       288Mi       3.3Gi
Swap:             0B          0B          0B

// 沒有成功的樣子
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       313Mi       1.9Gi       4.0Mi       1.6Gi       3.2Gi
Swap:          3.8Gi          0B       3.8Gi
```

2. 安裝apt相關套件
- ipvsadm: 集群使用 ipvs 的話要安裝
- nfs-common: 集群有要使用 nfs client 作為儲存的話要安裝
```bash
apt update -y
apt install -y net-tools wget curl unzip gnupg ipvsadm nfs-common
```


3. 於`/etc/sysctl.conf`，增加sysctl優化參數

```conf
tee -a /etc/sysctl.conf <<EOF
fs.nr_open = 10000000
fs.file-max = 11000000
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=524288
vm.overcommit_memory=1
kernel.pid_max=4194303
net.ipv4.ip_local_port_range=1024 65535
net.ipv4.tcp_slow_start_after_idle=0
net.ipv4.tcp_syn_retries=2
net.ipv4.tcp_synack_retries=2
net.ipv4.tcp_orphan_retries=3
net.ipv4.tcp_max_orphans=262144
net.ipv4.tcp_retries2=8
net.ipv4.tcp_max_syn_backlog=3240000
net.ipv4.tcp_keepalive_time=15
net.ipv4.tcp_keepalive_intvl=10
net.ipv4.tcp_keepalive_probes=9
net.ipv4.tcp_fin_timeout=15
net.ipv4.tcp_max_syn_backlog=3240000
EOF

sysctl -p
```

## 安裝CRI-O Runtime
### 設定crio.conf
```bash
cat <<EOF | tee /etc/modules-load.d/crio.conf
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
br_netfilter
overlay
EOF

modprobe -- overlay
modprobe -- br_netfilter
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack

cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
### 執行 `sysctl --system`
檢查模組是否存在，執行`lsmod | grep -e ip_vs -e nf_conntrack -e over，如果nf_conntrack模組啟動失敗，`執行重開機，執行`init 6 `

```
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 188416  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          188416  1 ip_vs
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
nf_defrag_ipv4         16384  1 nf_conntrack
libcrc32c              16384  2 nf_conntrack,ip_vs
overlay               163840  0
net_failover           24576  1 virtio_net
failover               16384  1 net_failover
```

### 安裝CRI-O Container Runtime
安裝版本建議要跟後續k8s套件的版本一樣，這份文件統一 v1.24，且一定要是`Debian_11`  
如果要安裝1.29之後的版本，這份文件的步驟完全不適用，請參考[官方教學](https://github.com/cri-o/cri-o/blob/main/install.md#installation-instructions)  

```bash
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/Debian_11/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.24/Debian_11/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:1.24.list

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.24/Debian_11/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/Debian_11/Release.key | apt-key add -

# install cri-o
apt-get update 
apt-get install -qq -y cri-o cri-o-runc cri-tools
```

重啟CRI-O，設定重啟後自動啟動

```bash
systemctl daemon-reload
systemctl enable --now crio
systemctl start crio
```

於`/usr/lib/systemd/system/crio.service`新增Insecure Registry 允許使用不安全的容器映像來源。

```conf
...
ExecStart=/usr/bin/crio \
          $CRIO_CONFIG_OPTIONS \
          $CRIO_RUNTIME_OPTIONS \
          $CRIO_STORAGE_OPTIONS \
          $CRIO_NETWORK_OPTIONS \
          $CRIO_METRICS_OPTIONS \
          --insecure-registry=<image repo address>
....
```

於`/etc/crio/crio.conf`新增CRI-O優化參數，否則k8s容器內不能用相關的network指令
```conf
[crio.runtime]
pids_limit = 4096
default_capabilities = [
"CHOWN",
"DAC_OVERRIDE",
"FSETID",
"FOWNER",
"SETGID",
"SETUID",
"SETPCAP",
"NET_BIND_SERVICE",
"KILL",
"NET_RAW",
]
```
重啟CRI-O

```bash
systemctl daemon-reload
systemctl restart crio
```


## Kubernetes元件
### 安裝Kubernetes元件 (版本要跟CRIO一樣)
(v1.24之後) 直接從官方repo安裝kubelet、kubeadm、kubectl，版本可替換 [官方指南](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

```bash
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gpg

# If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.24/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.24/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl

# 利用apt-mark功能，鎖定kubelet、kubeadm、kubectl的版本。
apt-mark hold kubelet kubeadm kubectl
# 開啟kubelet
systemctl enable kubelet
```

### **(master操作即可)** 初始化Kubernetes Cluster 
準備kubeadm-config.yaml 

```bash
cat > kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.24.0
networking:
  podSubnet: "10.244.0.0/16"
EOF
```

經由設定檔建立Kubernetes Cluster，執行指令`kubeadm init --config=kubeadm-config.yaml --upload-certs，注意預設會檢查master node cpu core >= 2，否則會出錯)`  
  
(重要)初始化完成會產生以下資訊，其中包含了「允許使用者操作Cluster的設定步驟」、「加入新Master需要的Token」、「加入新Node需要的Token」

```bash
...
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

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <master ip>:6443 --token dc8ke9.k86xx6xhjqftb1w4 \
        --discovery-token-ca-cert-hash sha256:12cca7b875fbf9e6cc3b551dfca282da3174d92cbe177a0ca2a5c8605596c762
```

確認所有kube-system的元件都正常啟動至Running狀態。

```
watch -n 1 kubectl get pod -n kube-system
```


### **(master操作即可)** 優化Kubernetes Cluster元件
注意事項: 發現static pod沒有立即重啟的話，`systemctl restart kubelet`  
1. Kuber-Proxy
調整kube-proxy的Routing原則，執行指令`kubectl edit configmap kube-proxy -n kube-system`並參照下列參考設定調整，! 表示要修改的地方
```yaml
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      ! strictARP: true
      syncPeriod: 0s
      tcpFinTimeout: 0s
      tcpTimeout: 0s
      udpTimeout: 0s
    kind: KubeProxyConfiguration
    metricsBindAddress: ""
    ! mode: "ipvs"
    nodePortAddresses: null
    oomScoreAdj: null
    portRange: ""
    showHiddenMetricsForVersion: ""
    udpIdleTimeout: 0s
    winkernel:
      enableDSR: false
      networkName: ""
      sourceVip: ""
```
重啟kube-proxy，執行指令`kubectl rollout restart ds/kube-proxy -n kube-system`

### 將節點(Worker)加入Kubernetes Cluster
在worker node執行上述產出的kubeadm join指令  
若是Token超過24hr而失效，可重新產出，在master node執行指令`kubeadm token create --print-join-command`  
### **(master操作即可)** 安裝CNI (Calico)
1. Kubernetes Cluster安裝完成之後，需要鋪設Container Network Interface(CNI)，啟動叢集內部虛擬網路的功能。用於Kubernetes Cluster剛初始化完成時。
2. 安裝Calico CNI時，應注意與Kubernetes的[版本匹配](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements)。
#### Calico 安裝步驟
執行指令`curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml -O`

修改calico.yaml
1. 將內部虛擬網路的網段由192.168改為10.244，執行指令
```bash
sed -i 's/# - name: CALICO_IPV4POOL_CIDR/- name: CALICO_IPV4POOL_CIDR/g' calico.yaml
sed -i 's/#   value: \"192\.168\.0\.0\/16\"/  value: \"10.244.0.0\/16\"/g' calico.yaml
```
2. (網卡不是eth0, 預設值的話) 修改網卡Primary參照，新增
IP_AUTODETECTION_METHOD的值更改為interface= **enp96s0f1**   
(原先只有IP:autodetect, IP_AUTODETECTION_METHOD要自行新增，如果沒改過網卡名稱，不設定也可以)
3. 執行指令`kubectl create -f calico.yaml`，啟動CNI。
4. 執行指令`kubectl get pod -n kube-system`，確定全部的node都有Calico Pod啟動並Running。
### **(Optional)** 安裝Consul & NFS client
如果有用到這兩個服務，記得另外在master & woker node 設定
### **(Optional)** kubectl 快速指令設定

vim `~/.bashrc`
```conf
alias k=kubectl
complete -o default -F __start_kubectl k
source <(kubectl completion bash)

#按tab就會自動完成resouces或是物件名稱, kubectl用k代替
k get pods
```

### **(Optional)** 安裝失敗/重裝cluster  
[TODO]



