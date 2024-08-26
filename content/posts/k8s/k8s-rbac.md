---
title: "Kubernetes RBAC Authorization"
date: 2024-08-27T02:32:00+08:00 
lastmod: 2024-08-27T02:32:00+08:00 
categories: 
- Kubernetes
tags: 
- Kubernetes
- RBAC
summary: "Kubernetes RBAC 權限控管筆記"
description: "Kubernetes RBAC 權限控管筆記" 
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
### [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/\#kubectl-create-clusterrolebinding)

{{< figure  src="rbac.png" title="k8s-rbac" width="70%" align="center">}}
- Assign some users have specific namespace/resources permissions with certain verbs  
- The RBAC API declares four kinds of Kubernetes objects: Role, ClusterRole, RoleBinding and ClusterRoleBinding.   
  You can check related resources with kubectl:
```bash
$ kubectl get clusterrole/role/clusterrolebinding/rolebinding
```
- #### Role v.s. ClusterRole

  - An RBAC Role or ClusterRole contains rules that represent a set of permissions. Permissions are purely additive (there are no "deny" rules).  
  - A Role always sets permissions within a particular namespace; when you create a Role, you have to specify the namespace it belongs in.  
  - ClusterRole, by contrast, is a non-namespaced resource. The resources have different names (Role and ClusterRole) because a Kubernetes object always has to be either namespaced or not namespaced; it can't be both.


- #### RoleBinding and ClusterRoleBinding 

  - A role binding grants the permissions defined in a role to a user or set of users (users, groups, or service account).  
  - A RoleBinding may reference any Role in the same namespace. Alternatively, a RoleBinding can reference a ClusterRole and bind that ClusterRole to the namespace of the RoleBinding.   
  - If you want to bind a ClusterRole to all the namespaces in your cluster, you use a ClusterRoleBinding.

## 新增k8s使用者
1. ### 創建憑證 
```bash
cd /etc/kubernetes/pki #k8s相關憑證都在這裡 
(umask 077; openssl genrsa -out reader.key 2048)
```
CN 要跟後面指令/yaml中的 User 保持一致
```bash
openssl req -new -key reader.key -out reader.csr -subj "/CN=reader"
```
使用 kubeadm ca憑證簽發新憑證 證書有效十年
```bash
openssl x509 -req -in reader.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out reader.crt -days 3650

# 輸出如下
Signature ok
subject=/CN=reader
Getting CA Private Key
```
  
(TODO: organize the format)  
2. ### 建立role/clusterrole/bindings

   1. #### 自定義角色 [官方文件範例](https://kubernetes.io/docs/reference/access-authn-authz/rbac/\#role-example)

      建立yaml檔，用kubectl apply \-f \<rabc\_file\_name\>.yaml 建立角色

- create role

| apiVersion: rbac.authorization.k8s.io/v1 kind: Role metadata:  namespace: default *\# role要指定ns*  name: pod-reader rules: \- apiGroups: \[""\] *\# "" indicates the core API group*  resources: \["pods"\]  verbs: \["get", "watch", "list"\] |
| :---- |


- create rolebinding [官方文件範例](https://kubernetes.io/docs/reference/access-authn-authz/rbac/\#rolebinding-and-clusterrolebinding)

| apiVersion: rbac.authorization.k8s.io/v1 kind: RoleBinding metadata:  name: read-pods  namespace: default subjects: \- kind: User  name: jane *\# "name" is case sensitive, 要跟產生key的CN一樣*  apiGroup: rbac.authorization.k8s.io roleRef:  kind: Role *\#this must be Role or ClusterRole*  name: pod-reader *\# 要跟role name一樣*  apiGroup: rbac.authorization.k8s.io |
| :---- |

  2. #### [使用CLI快速建立](https://kubernetes.io/docs/reference/access-authn-authz/rbac/\#command-line-utilities)RBAC

     使用kubectl get 可以看到[預設](https://kubernetes.io/docs/reference/access-authn-authz/rbac/\#default-roles-and-role-bindings)或是已建立的role/clusterRole

| kubectl get clusterrole/role/clusterrolebinding/rolebinding |
| :---- |

     假設要建立一個對cluster 只能讀取的角色(預設的clusterRole, view)，注意user要跟產生key的CN一樣

| kubectl create clusterrolebinding root-cluster-view-binding \--clusterrole=view \--user=reader |
| :---- |

     

     (假設2) 要建立一個對cluster 有全部kubectl權限的角色

     i. 直接配置/etc/kubernetes/admin.conf 到指定user

			ii. binding cluster-admin 的 clusterRole

| kubectl create clusterrolebinding root-cluster-admin-binding \--clusterrole=cluster-admin \--user=new\_admin |
| :---- |

3. ### 為user創建 KUBECONFIG 配置文件

1. 設定cluster

   找到controlPlaneEndpoint

| $ kubectl config view apiVersion: v1 clusters: \- cluster:     certificate-authority-data: DATA+OMITTED     server: https://172.16.199.236:8443   name: kubernetes |
| :---- |

   替換紅底處 

| $ kubectl config set-cluster kubernetes \\   \--server=https://172.16.199.236:8443  \\   \--certificate-authority=/etc/kubernetes/pki/ca.crt \\   \--embed-certs=true \\   \--kubeconfig=/tmp/reader.config \# user 要用的KUBECONFIG 存放處 Cluster "kubernetes" set. |
| :---- |

   

2. 設定憑證

   紅底處要換成生成憑證/存放config的位置

| $kubectl config set-credentials reader \\   \--client-certificate=/etc/kubernetes/pki/reader.crt \\   \--client-key\=/etc/kubernetes/pki/reader.key \\   \--embed-certs=true \\   \--kubeconfig=/tmp/reader.config User "reader" set. |
| :---- |

   

3. 設定context，注意如果是role的話要特別指定`--namespace`

| $kubectl config set-context reader@kubernetes \\   \--cluster kubernetes \\   \--user=reader \\   \--kubeconfig=/tmp/reader.config Context "reader@kubernetes" created. |
| :---- |

4. 測試&輸出config，打kubectl相關指令看是不是理想的權限

| kubectl config use-context reader@kubernetes \\   \--kubeconfig=/tmp/reader.config |
| :---- |

5. 將做好的kubeconfig複製到目標使用者的\~/.kube/config，注意要開啟讀取權限，並切換context

| mkdir \-p $HOME/.kube sudo cp \-i /tmp/reader.config $HOME/.kube/config sudo chown $(id \-u):$(id \-g) $HOME/.kube/config cd $HOME/.kube kubectl config use-context reader@kubernetes \--kubeconfig=config kubectl config view \#確認目前context, user 輸出如下: apiVersion: v1 clusters: \- cluster:     certificate-authority-data: DATA+OMITTED     server: https://172.16.199.236:8443   name: kubernetes contexts: \- context:     cluster: kubernetes     user: reader   name: reader@kubernetes current-context: reader@kubernetes kind: Config preferences: {} users: \- name: reader   user:     client-certificate-data: REDACTED     client-key-data: REDACTED |
| :---- |

6. 查看特定user的權限

| $ kubectl auth can-i \--list \--as=\<username\> |
| :---- |

   

## 參考

### 課程

[https://hiskio.com/en/courses/349/lectures/184577?utm\_source=google\&utm\_medium=cpc\&utm\_campaign=hiskio\_autosale\&utm\_term=hiskio\_autosale](https://hiskio.com/en/courses/349/lectures/184577?utm\_source=google\&utm\_medium=cpc\&utm\_campaign=hiskio\_autosale\&utm\_term=hiskio\_autosale)

[https://hiskio.com/en/courses/440/lectures/22270](https://hiskio.com/en/courses/440/lectures/22270)

[Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/\#explore-the-home-kube-directory)  
[Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)  
[Assign permissions to an user in Kubernetes. An overview of RBAC-based AuthZ in k8s](https://faun.pub/assign-permissions-to-an-user-in-kubernetes-an-overview-of-rbac-based-authz-in-k8s-7d9e5e1099f1)  
[Kubernetes 创建普通账号](https://huangzhongde.cn/post/Kubernetes/Kubernetes%E5%88%9B%E5%BB%BA%E6%99%AE%E9%80%9A%E8%B4%A6%E5%8F%B7/)  
[從異世界歸來的第二九天 \- Kubernetes Security (二) \- RBAC Authorization 權限管理](https://ithelp.ithome.com.tw/articles/10300806)  
[官方RBAC中文版整理+翻譯](https://tn710617.github.io/zh-tw/kubernetes-using-rbac-authorization/)  
[kubectl config set-cluster](https://kubernetes.io/docs/reference/kubectl/generated/kubectl\_config/kubectl\_config\_set-cluster/)  
[k8s集群外主机通过kubectl访问集群](https://www.cnblogs.com/zuoyang/p/15261373.html)  
[鳥哥: Linux 帳號管理與 ACL 權限設定](https://linux.vbird.org/linux\_basic/centos7/0410accountmanager.php)
