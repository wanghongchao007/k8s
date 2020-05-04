# 1.docker制作本地镜像并上传到远端仓库
```shell
[root@wanghongchao-jenkins ~]# docker  commit 0b59dab18fd9  yczxvf/whc
sha256:e91c702ff95c25b8510a33510a1f3ffcd3edefe3c30ffe5881bdcccc3391da80
[root@wanghongchao-jenkins ~]# docker  ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS              PORTS                                              NAMES
0b59dab18fd9        jenkinsci/blueocean:1.22.0   "/sbin/tini -- /usr/…"   4 days ago          Up 13 minutes       0.0.0.0:8080->8080/tcp, 0.0.0.0:50000->50000/tcp   jenkins-whc
[root@wanghongchao-jenkins ~]# docker  commit 0b59dab18fd9  yczxvf/whc
sha256:e91c702ff95c25b8510a33510a1f3ffcd3edefe3c30ffe5881bdcccc3391da80
[root@wanghongchao-jenkins ~]# docker  images
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
yczxvf/whc            latest              e91c702ff95c        35 seconds ago      569MB
<none>                <none>              280769e21eb0        28 hours ago        204MB
jenkinsci/blueocean   1.22.0              029d04659fd8        5 days ago          566MB
centos                7.7.1908            08d05d1d5859        5 months ago        204MB
[root@wanghongchao-jenkins ~]# docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: yczxvf 
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@wanghongchao-jenkins ~]# docker push yczxvf/whc:latest 
The push refers to repository [docker.io/yczxvf/whc]
bb32517f766b: Pushed 
284099e1bb9d: Mounted from jenkinsci/blueocean 
aad218152f64: Mounted from jenkinsci/blueocean 
62db8b140792: Mounted from jenkinsci/blueocean 
2feef26d82d2: Mounted from jenkinsci/blueocean 
4ee3fb827326: Mounted from jenkinsci/blueocean 
81867172e638: Mounted from jenkinsci/blueocean 
ec9db2b45d62: Mounted from jenkinsci/blueocean 
fd0d47299cea: Mounted from jenkinsci/blueocean 
465d1152f666: Mounted from jenkinsci/blueocean 
a833b1d6f289: Mounted from jenkinsci/blueocean 
9e6598bb756d: Mounted from jenkinsci/blueocean 
276165b607bf: Mounted from jenkinsci/blueocean 
ceaf9e1ebef5: Mounted from jenkinsci/blueocean 
9b9b7f3d56a0: Mounted from jenkinsci/blueocean 
f1b5933fe4b5: Mounted from jenkinsci/blueocean 
latest: digest: sha256:71f95e52c9e19e859b6b69475fa2a95903485d256c9d4fcde48146b85099e296 size: 3668
[root@wanghongchao-jenkins ~]# docker inspect yczxvf/whc:latest 
[
    {
        "Id": "sha256:e91c702ff95c25b8510a33510a1f3ffcd3edefe3c30ffe5881bdcccc3391da80",
        "RepoTags": [
            "yczxvf/whc:latest"
        ],
        "RepoDigests": [
...................
...................
...................
        "Metadata": {
            "LastTagTime": "2020-04-23T16:00:16.588685188+08:00"
        }
    }
]
[root@wanghongchao-jenkins ~]# 

```

# 2.docker上传本地镜像到远端仓库
```nginx
[root@k8s-node-1 ~]# docker  login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: yczxvf
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@k8s-node-1 ~]# docker  images
REPOSITORY                                           TAG                 IMAGE ID            CREATED             SIZE
kubernetesui/dashboard                               v2.0.0              8b32422733b3        12 days ago         222MB
kubernetesui/metrics-scraper                         v1.0.4              86262685d9ab        5 weeks ago         36.9MB
quay-mirror.qiniu.com/coreos/flannel                 v0.12.0-amd64       4e9f801d2217        7 weeks ago         52.8MB
registry.aliyuncs.com/google_containers/kube-proxy   v1.16.0             c21b0c7400f9        7 months ago        86.1MB
jettech/kube-webhook-certgen                         v1.0.0              f3b9b9fa842c        9 months ago        51.5MB
registry.aliyuncs.com/google_containers/pause        3.1                 da86e6ba6ca1        2 years ago         742kB
nginx                                                1.7.9               84581e99d807        5 years ago         91.7MB
[root@k8s-node-1 ~]# docker tag  kubernetesui/dashboard:v2.0.0  yczxvf/dashboard:v2.0.0
[root@k8s-node-1 ~]# docker tag kubernetesui/metrics-scraper:v1.0.4  yczxvf/metrics-scraper:v1.0.4
[root@k8s-node-1 ~]# docker push yczxvf/metrics-scraper:v1.0.4
[root@k8s-node-1 ~]# docker push  yczxvf/dashboard:v2.0.0 
```