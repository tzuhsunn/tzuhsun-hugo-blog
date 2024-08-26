---
title: "Kubernetes Single Master Cluster Install (CRI-O, Debian 12, >v1.24)"
date: 
lastmod: 
categories: 
- Kubernetes
tags: 
- Kubernetes
- Debian
- disaster
summary: "重新安裝/設定 K8S"
description: "重新安裝/設定 K8S" 
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
* 此份為記錄重設 k8s cluster 過程中遇到的問題及踩坑
* 此為Kubernetes Cluster安裝文件，只有單個master，適用於Debian12，所有步驟worker, master都要執行，除了 **(Optional)** 跟 **(master操作即可)**。
* 安裝 Kubernetes Cluster 可參考 [Kubernetes Single Master Cluster Install](https://tzuhsun-blog.web.app/posts/k8s/k8s-install/)

# 操作步驟
## 在 worker node 執行 kubeadm reset
從worker node 開始執行指令
```bash
$ kubeadm reset -f
```
如果有任何指示要照做，例如
- master node 要移除 `$HOME/.kube/config file`
- `If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)``
- remove /etc/cni/net.d 中calico相關檔案 (不要刪掉crio或是CRI相關檔案，否則會需要重裝crio，會哭)
- (如果有安裝CNI) 刪掉 calico 網卡 `modprobe -r ipip` [calico清除相關](https://qiaolb.github.io/remove-calico.html)

## 在 controlplane 刪除 worker node / 非 master 的 controlplane
記得留一個 mater node 執行 `$ kubectl delete node <node_name>`
最後再 `$ kubeadm reset -f`

## (如果是安裝 K8S with HA) 重新初始化
照著文件重新架設 HA cluster

# 問題
不小心把所有資源刪除，致使 kube-system namespace 底下沒有 kube-proxy, core-dns, calico等服務

1. [復原kube-proxy](https://stackoverflow.com/questions/71264643/how-to-restore-accidentally-deleted-a-kube-proxy-daemonset-in-a-kubernetes-clust)
```bash
$ kubeadm init phase addon kube-proxy --kubeconfig ~/.kube/config
```

2. [復原 core-dns](https://blog.csdn.net/weixin_41831919/article/details/119980118)
在CNI活著/沒有另外設定CNI的前提下，可以直接從 core-dns 官方 repo 重新下載 
```bash
wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed
wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/deploy.sh
# 安裝jquery
yum install -y jq
chmod +x deploy.sh
./deploy.sh -i 10.96.0.10 > coredns.yml
kubectl apply -f coredns.yml
```
[core-dns和CNI(calico)的關係](https://blog.csdn.net/weixin_60712169/article/details/123553457)



