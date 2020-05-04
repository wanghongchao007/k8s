# 加入节点报错

### 1.[WARNING IsDockerSystemdCheck]: detected \"cgroupfs\" as the Docker cgroup driver. The recommended driver is \"systemd\". Please follow the guide at https://kubernetes.io/docs/setup/cri/\nerror execution phase preflight: [preflight] Some fatal errors occurred:\n\t
```shell
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
# Restart Docker
systemctl daemon-reload
systemctl restart docker
```
### 2.error execution phase preflight: couldn't validate the identity of the API Server: abort connecting to API servers after timeout of 5m0s
To see the stack trace of this error execute with --v=5 or higher
```shell
# 2.1.kubeadm token create --print-join-command

##2.1.在master重新生成token
# kubeadm token create
424mp7.nkxx07p940mkl2nd
# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
5301764d63000827d078f56c419abeab2380e4317b9fdd60b660cd70eba08d9a
Node节点重新join就可以了
kubeadm join 192.168.150.60:6443 --token w2gusi.zsglyb1q2m1wrdj6     --discovery-token-ca-cert-hash sha256:5301764d63000827d078f56c419abeab2380e4317b9fdd60b660cd70eba08d9a
````
### 3. [ERROR DirAvailable–var-lib-etcd]: /var/lib/etcd is not empty 
```shell
rm -rf /var/lib/etcd
````
### 4.ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1\n
```shell
echo '1' > /proc/sys/net/ipv4/ip_forward
````
### 5.ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1
```shell
echo "1" >/proc/sys/net/bridge/bridge-nf-call-iptables
````

- https://cloud.tencent.com/developer/article/1461571
- https://kubernetes.io/docs/setup/production-environment/container-runtimes/