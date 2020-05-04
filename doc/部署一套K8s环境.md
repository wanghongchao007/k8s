# 部署一套K8s环境
##  一、准备服务器节点
### 服务器节点IP（hostname）：
```shell
192.168.150.60 k8s-master
192.168.150.61 k8s-node-1
192.168.150.62 k8s-node-2
192.168.150.63 k8s-node-3
```
### 操作系统版本
- cat /etc/redhat-release: ***CentOS Linux release 7.6.1810 (Core)*** 
- uname  -a: ***3.10.0-957.el7.x86_64***
## 二. 配置Ansible
### 1.在Ansible服务器上的/etc/hosts文件中添加k8s服务器节点信息
```shell
[root@ansible ~]# vim /etc/hosts
[root@ansible ~]# cat  /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.150.60 k8s-master
192.168.150.61 k8s-node-1
192.168.150.62 k8s-node-2
192.168.150.63 k8s-node-3
```
### 2.在Ansible服务器上的/etc/ansible/hosts文件中添加k8s服务器节点
```shell
[root@ansible ~]# vim  /etc/ansible/hosts 
[root@ansible ~]# grep  -v '^#' /etc/ansible/hosts | grep -v '^$'
[k8s-all]
192.168.150.60 
192.168.150.61
192.168.150.62
192.168.150.63
[k8s-master]
k8s-master
[k8s-nodes]
k8s-node-1
k8s-node-2
k8s-node-3
```
## 三. 修改k8s集群各节点/etc/hosts
### 1.创建playbook文件（参考 set_hosts_playbook.yml）
```shell
[root@ansible ~]# cat  set_hosts_playbook.yml
---
- hosts: k8s-all
  remote_user: root

  tasks:
    - name: backup /etc/hosts
      shell: mv /etc/hosts /etc/hosts_bak

    - name: copy local hosts file to remote
      copy: src=/etc/hosts dest=/etc/ owner=root group=root mode=0644 
```
### 2.执行ansible-playbook
```shell
[root@ansible ~]# ansible-playbook set_hosts_playbook.yml
```
![hosts文件.png](https://ae03.alicdn.com/kf/Hfd8f900c37054461ba41c0386d2ee585q.png)

## 四. 安装Docker

- 在所有主机上安装Docker
### 1.创建playbook文件（参考 install_docker_playbook.yml）
```shell
[root@ansible ~]# vim  install_docker_playbook.yml
[root@ansible ~]# cat  install_docker_playbook.yml 
- hosts: k8s-all
  remote_user: root
  vars: 
     docker_version: 18.09.2

  tasks: 
     - name: install dependencies
       #shell: yum install -y yum-utils device-mapper-persistent-data lvm2 
       yum: name={{item}} state=present
       with_items:
          - yum-utils
          - device-mapper-persistent-data
          - lvm2

     - name: config yum repo
       shell: yum-config-manager --add-repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo

     - name: install docker
       yum: name=docker-ce-{{docker_version}} state=present

     - name: start docker
       shell: systemctl enable docker && systemctl start docker
```
### 2.执行ansible-playbook
```shell
[root@ansible ~]# vim  /etc/ansible/ansible.cfg
deprecation_warnings = false  ## 179默认是true，并且不生效
[root@ansible ~]# ansible-playbook install_docker_playbook.yml
```
![Docker安装.png](https://ae04.alicdn.com/kf/Hd2ab0186966c423eae801906ad628073L.png)

## 五. 部署k8s master

### 1.开始部署之前，需要做一些初始化处理：关闭防火墙、关闭selinux、禁用swap、配置k8s阿里云yum源等，所有操作放在脚本 pre-setup.sh 中，并在2中playbook中通过script模块执行
```shell
[root@ansible ~]# cat  pre-setup.sh 
#!/bin/bash
#关闭防火墙
systemctl disable firewalld
systemctl stop firewalld
#关闭selinux
setenforce 0
#永久关闭
sed -i "s/SELINUX=enforcing/SELINUX=disabled/" /etc/sysconfig/selinux
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
#禁用swap
swapoff -a
#永久禁用
sed -i "s/.*swap.*/#&/" /etc/fstab
#修改内核参数
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
#重新加载配置文件
sysctl --system
#配置阿里k8s yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
#更新缓存
yum clean all -y && yum makecache -y && yum repolist -y
[root@ansible ~]# chmod  a+x pre-setup.sh
```
### 2.创建playbook文件 deploy_master_playbook.yml，只针对master节点，安装kubectl，kubeadm，kubelet，以及flannel（将kube-flannel.yml文件里镜像地址的quay.io改为quay-mirror.qiniu.com避免超时，参考 kube-flannel.yml）

- wget  -c   https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml

```shell
[root@ansible ~]# vim  deploy_master_playbook.yml
[root@ansible ~]# cat  deploy_master_playbook.yml 
- hosts: k8s-master
  remote_user: root
  vars:
    kube_version: 1.16.0-0
    k8s_version: v1.16.0
    k8s_master: 192.168.150.60
  tasks:
    - name: prepare env
      script: ./pre-setup.sh      

    - name: install kubectl,kubeadm,kubelet
      yum: name={{item}} state=present
      with_items:
        - kubectl-{{kube_version}}
        - kubeadm-{{kube_version}}
        - kubelet-{{kube_version}}

    - name: init k8s
      shell: kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version {{k8s_version}} --apiserver-advertise-address {{k8s_master}}  --pod-network-cidr=10.244.0.0/16 --token-ttl 0

    - name: config kube
      shell: mkdir -p $HOME/.kube && cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && chown $(id -u):$(id -g) $HOME/.kube/config

    - name: copy flannel yaml file
      copy: src=./kube-flannel.yml dest=/tmp/ owner=root group=root mode=0644 

    - name: install flannel
      shell: kubectl apply -f /tmp/kube-flannel.yml

    - name: get join command
      shell: kubeadm token create --print-join-command 
      register: join_command
    - name: show join command
      debug: var=join_command verbosity=0
```
### 3.执行ansible-playbook
```shell
[root@ansible ~]# vim  /etc/ansible/ansible.cfg 
command_warnings = False   ### 187行
[root@ansible ~]# ansible-playbook deploy_master_playbook.yml
```
### 4.结果回显

![3.png](https://ae03.alicdn.com/kf/H84b5b45e045541298a7d9585ce9e5e60h.png)

```shell
[root@k8s-master ~]# docker  images
REPOSITORY                                                        TAG                 IMAGE ID            CREATED             SIZE
quay-mirror.qiniu.com/coreos/flannel                              v0.12.0-amd64       4e9f801d2217        7 weeks ago         52.8MB
registry.aliyuncs.com/google_containers/kube-apiserver            v1.16.0             b305571ca60a        7 months ago        217MB
registry.aliyuncs.com/google_containers/kube-proxy                v1.16.0             c21b0c7400f9        7 months ago        86.1MB
registry.aliyuncs.com/google_containers/kube-controller-manager   v1.16.0             06a629a7e51c        7 months ago        163MB
registry.aliyuncs.com/google_containers/kube-scheduler            v1.16.0             301ddc62b80b        7 months ago        87.3MB
registry.aliyuncs.com/google_containers/etcd                      3.3.15-0            b2756210eeab        8 months ago        247MB
registry.aliyuncs.com/google_containers/coredns                   1.6.2               bf261d157914        8 months ago        44.1MB
registry.aliyuncs.com/google_containers/pause                     3.1                 da86e6ba6ca1        2 years ago         742kB
[root@k8s-master ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-58cc8c89f4-bt86l             1/1     Running   0          17h
kube-system   coredns-58cc8c89f4-tdqsg             1/1     Running   0          17h
kube-system   etcd-k8s-master                      1/1     Running   0          17h
kube-system   kube-apiserver-k8s-master            1/1     Running   0          17h
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          17h
kube-system   kube-flannel-ds-amd64-v57pg          1/1     Running   0          17h
kube-system   kube-proxy-xtjkl                     1/1     Running   0          17h
kube-system   kube-scheduler-k8s-master            1/1     Running   0          17h
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   17h   v1.16.0
```

## 六. 部署k8s node

### 1.同master一样，开始部署之前，需要做一些初始化处理：关闭防火墙、关闭selinux、禁用swap、配置k8s阿里云yum源等，所有操作放在脚本  pre-setup.sh 中，并在2中playbook中通过script模块执行
### 2.创建playbook文件 deploy_nodes_playbook.yml，针对除master外的其它集群节点，安装kubeadm，kubelet，并将节点加入到k8s集群中，使用的是前面部署master时输出的加入集群命令
```shell
[root@ansible ~]# cat  deploy_nodes_playbook.yml 
- hosts: k8s-nodes
  remote_user: root
  vars:
     kube_version: 1.16.0-0
  tasks:
    - name: prepare env
      script: ./pre-setup.sh

    - name: install kubeadm,kubelet
      yum: name={{item}} state=present
      with_items:
        - kubeadm-{{kube_version}}
        - kubelet-{{kube_version}}

    - name: start kubelt
      shell: systemctl enable kubelet && systemctl start kubelet

    - name: join cluster
      shell: kubeadm join 192.168.150.60:6443 --token pimzjc.hretsg56jgexiulm     \
      --discovery-token-ca-cert-hash sha256:f1db4ed050a7dc87d6e8395fae7673dbc5bf73c873e99ca2e8ff6d561e36c5ed
```
### 3.执行ansible-playbook
```shell
[root@ansible ~]# ansible-playbook deploy_nodes_playbook.yml
```
### 4.在master节点上通过kubectl get nodes看到加入到集群中的节点，并且status为Ready状态

```shell
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   33m     v1.16.0
k8s-node-1   Ready    <none>   8m39s   v1.16.0
k8s-node-2   Ready    <none>   2m7s    v1.16.0
k8s-node-3   Ready    <none>   2m11s   v1.16.0
[root@k8s-master ~]# kubectl get cs -o=go-template='{{printf "|NAME|STATUS|MESSAGE|\n"}}{{range .items}}{{$name := .metadata.name}}{{range .conditions}}{{printf "|%s|%s|%s|\n" $name .status .message}}{{end}}{{end}}'
|NAME|STATUS|MESSAGE|
|controller-manager|True|ok|
|scheduler|True|ok|
|etcd-0|True|{"health":"true"}|
```

## 七. 安装Ingress
- Ingress为集群内服务提供外网访问，包括基于Nginx与Traefik两个版本，这里使用比较熟悉的Nginx版本。安装Ingress的操作在master节点进行（因为前面在master节点安装并配置了kubectl，也可在其它安装并配置好了kubectl的节点进行）
### 1.下载yaml文件
```shell
[root@k8s-master ~]# wget -O nginx-ingress.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
```
- 将里面的quay.io修改为quay-mirror.qiniu.com，避免镜像拉取超时。同时在nginx-ingress-controller的Deployment上添加hostNetwork为true及nginx-ingress的标签，以使用宿主机网络与控制Ingress部署的节点
```shell
[root@k8s-master ~]# cat  nginx-ds.yml 
apiVersion: v1
kind: Service
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  type: NodePort
  selector:
    app: nginx-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: nginx-ds
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
### 2.部署Ingress
- 首先在knode1节点上打标签nginx-ingress=true，控制Ingress部署到knode1上，保持IP固定。
-  https://blog.csdn.net/wangshuminjava/article/details/92619413 
- 然后完成nginx-ingress的部署
- 部署完成，稍等片刻等Pod创建完成，可通过如下命令查看ingress相关Pod情况
```shell
[root@k8s-master ~]# kubectl apply -f nginx-ds.yml 
service/nginx-ds created
daemonset.apps/nginx-ds created
[root@k8s-master ~]# kubectl get pods -o wide | grep nginx-ds
nginx-ds-cmdnb   1/1     Running   0          153m   10.244.1.18   k8s-node-1   <none>           <none>
nginx-ds-dz2gh   1/1     Running   0          153m   10.244.3.7    k8s-node-2   <none>           <none>
nginx-ds-j6kqp   1/1     Running   0          153m   10.244.2.7    k8s-node-3   <none>           <none>
[root@k8s-master ~]# kubectl get services -o wide 
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE    SELECTOR
kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP        21h    <none>
nginx-ds     NodePort    10.110.2.84   <none>        80:31116/TCP   153m   app=nginx-ds
```
### 3.访问测试
```shell
[root@k8s-master ~]# curl  10.110.2.84
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
![nginx.png](https://i.loli.net/2020/05/04/snelz1ktFGCJxBa.png)

## 八. 安装Kubernetes Dashboard

### 1. 到github地址上一步步找:https://github.com/kubernetes/dashboard/tree/v2.0.0-beta4/aio/deploy下的recommended.yaml文件; 

-  https://zhuanlan.zhihu.com/p/91731765 

```shell
[root@k8s-master ~]# ll recommended.yaml 
-rw-r--r-- 1 root root 7552 4月  21 21:10 recommended.yaml
```
### 2. 执行kubectl apply -f recommended.yaml; 
```shell
[root@k8s-master ~]# kubectl apply -f recommended.yaml
```
### 3. 部署Dashboard
```shell
[root@k8s-master ~]# kubectl get pods -n kube-system
NAME                                 READY   STATUS    RESTARTS   AGE
coredns-58cc8c89f4-ckndx             1/1     Running   0          79m
coredns-58cc8c89f4-w6zf6             1/1     Running   0          55m
etcd-k8s-master                      1/1     Running   0          78m
kube-apiserver-k8s-master            1/1     Running   0          78m
kube-controller-manager-k8s-master   1/1     Running   0          78m
kube-flannel-ds-amd64-22h89          1/1     Running   1          48m
kube-flannel-ds-amd64-pfkvr          1/1     Running   0          55m
kube-flannel-ds-amd64-pfwsd          1/1     Running   0          55m
kube-flannel-ds-amd64-v6ssq          1/1     Running   0          48m
kube-proxy-qrqng                     1/1     Running   0          48m
kube-proxy-wqwt6                     1/1     Running   0          79m
kube-proxy-wvd72                     1/1     Running   0          48m
kube-proxy-z5zhk                     1/1     Running   0          55m
kube-scheduler-k8s-master            1/1     Running   0          78m
[root@k8s-master ~]# kubectl get pods --namespace=kubernetes-dashboard -o wide
NAME                                        READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
dashboard-metrics-scraper-c79c65bb7-grr5g   1/1     Running   0          66s   10.244.1.27   k8s-node-1   <none>           <none>
kubernetes-dashboard-56484d4c5-9hlvz        1/1     Running   0          66s   10.244.1.28   k8s-node-1   <none>           <none>
[root@k8s-master ~]# kubectl --namespace=kubernetes-dashboard get service kubernetes-dashboard
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes-dashboard   ClusterIP   10.107.120.156   <none>        443/TCP   2m25s
# 编辑kubernetes-dashboard：kubectl --namespace=kubernetes-dashboard edit service kubernetes-dashboard，将里面的type: ClusterIP改为type: NodePort即可。
wq保存即可。等一会儿，重新查看，就变为NodePort了。
[root@k8s-master ~]# kubectl --namespace=kubernetes-dashboard edit service kubernetes-dashboard
service/kubernetes-dashboard edited
[root@k8s-master ~]# kubectl --namespace=kubernetes-dashboard get service kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.107.120.156   <none>        443:31002/TCP   8m46s
```
### 4.访问Dashboard
- 访问 https://集群任意节点IP:30443，打开Dashboard登录页面，执行如下命令获取登录token
![Dashboard-1.png](https://s1.ax1x.com/2020/05/04/YC0MqS.png)

## 九. 解决证书无效问题

- 安装完后，默认的证书可能无效，在Chrome浏览中无法打开Dashboard，可通过重新生成证书解决。

### 1.创建自定义证书
```shell
#新建目录：
[root@k8s-master ~]# mkdir key && cd key
# 生成私钥证书
[root@k8s-master key]# openssl genrsa -out dashboard.key 2048 
Generating RSA private key, 2048 bit long modulus
...................+++
.........................................................+++
e is 65537 (0x10001)
#我这里写的自己的node1节点，因为我是通过nodeport访问的；如果通过apiserver访问，可以写成自己的master节点ip
[root@k8s-master key]# openssl req -new -out dashboard.csr -key dashboard.key -subj '/CN=192.168.150.61'
# 使用集群的CA来签署证书
[root@k8s-master key]# openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt 
Signature ok
subject=/CN=192.168.150.61
Getting Private key
#删除原有的证书secret
[root@k8s-master key]# kubectl delete secret kubernetes-dashboard-certs -n kubernetes-dashboard
secret "kubernetes-dashboard-certs" deleted
#创建新的证书secret
[root@k8s-master key]# kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt -n kubernetes-dashboard
secret/kubernetes-dashboard-certs created
#查看pod
[root@k8s-master key]# kubectl get pod -n kubernetes-dashboard
NAME                                        READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-c79c65bb7-grr5g   1/1     Running   0          15m
kubernetes-dashboard-56484d4c5-9hlvz        1/1     Running   0          15m
#重启pod
[root@k8s-master key]# kubectl  delete pod  kubernetes-dashboard-56484d4c5-9hlvz  -n kubernetes-dashboard
pod "kubernetes-dashboard-56484d4c5-9hlvz" deleted
[root@k8s-master key]# kubectl --namespace=kubernetes-dashboard get service kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.107.120.156   <none>        443:31002/TCP   27m
[root@k8s-master key]# kubectl get pods --namespace=kubernetes-dashboard -o wide
NAME                                        READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
dashboard-metrics-scraper-c79c65bb7-grr5g   1/1     Running   0          28m   10.244.1.27   k8s-node-1   <none>           <none>
kubernetes-dashboard-56484d4c5-d5rqf        1/1     Running   0          11m   10.244.1.29   k8s-node-1   <none>           <none>
```
### 2. 新建用户获取令牌
[![Dashboard-1.png](https://s1.ax1x.com/2020/05/04/YC0OL8.png)](https://imgchr.com/i/YC0OL8)
#### 2.1.新建用户
```shell
[root@k8s-master ~]# cat  admin-user.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
[root@k8s-master ~]# kubectl  apply -f  admin-user.yaml 
serviceaccount/admin-user created
```
#### 2.2.绑定用户关系
```shell
[root@k8s-master ~]# cat  admin-user-role-binding.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
[root@k8s-master ~]# kubectl  apply -f  admin-user-role-binding.yaml 
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```
#### 2.3.获取令牌
```shell
[root@k8s-master ~]# kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-lmrcc
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 0c975ada-dbea-4139-82ac-8e7bd311e723

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Inc0T19KUldOakRKODZSdThFMHZOQjJjSHcxcEI5OXE3ZUt6UFF3cEdEYUUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWxtcmNjIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIwYzk3NWFkYS1kYmVhLTQxMzktODJhYy04ZTdiZDMxMWU3MjMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.ZryYCLrhJVR-H7qsCQNPvqSniATyrZYM1ApTpEt7xbsHiBC7JvICFfuluEBg7rTcOeDY-ZpmLDFApMsYuyOiscXjlxA5os6LBulu7WFrKk65PzRUeO86kuRrJIHl2VjJVZmWBWqvC76cMIu6SBERbNoRAe-gV7z9N4rrFVB68xnpS_mdQ-dUk8Xj7GympbTLJoKY2QiP15RnDbH_eqb6FcybhXmGVmRjzxjBxkkuVB_7KBkaGy6vBNS2ZiQa78es8WDdfU7_MbXDaBwXDm1uDvHE0we0gdvDTJPpRSLsl3f32WcdpRxLU5WqLZZwz-2XUsEMohGSjVvanr4sT-vYlQ
```
#### 2.4.输入令牌，访问测试
![Dashboard-3.png](https://s1.ax1x.com/2020/05/04/YC0lVg.png)
![Dashboard-4.png](https://s1.ax1x.com/2020/05/04/YC0arT.png)

![Dashboard-5.png](https://s1.ax1x.com/2020/05/04/YCBQQx.png)
![Dashboard-6.png](https://s1.ax1x.com/2020/05/04/YCBMS1.png)

