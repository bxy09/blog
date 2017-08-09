---
layout: "post"
title: "kubernetes在中国墙内启动并且适应Systemd"
date: 2016-02-02 10:48:00
tags: kubernetes, docker, 云, systemd
---
该配置适用于k8s v1.1版本，其他版本没有测试过，基本参照自ubuntu的配置方式（感谢浙大同学们的工作），
主要的区别在于将一些docker镜像从Google的源更换到国内的daocloud.io，以及适配Debian系统所用的Systemd服务模块(取代ubuntu中的upstart).

基本参照于[k8s deployment on bare-metal ubuntu nodes](http://kubernetes.io/v1.1/docs/getting-started-guides/ubuntu.html), 以下只介绍需要参与的部分以及修改的部分。

最终工程为：https://github.com/bxy09/k8s-debian

在debian jessie下进行了测试，其他debian版本或者其他发行版请酌情修改。

<!--more-->

```
git clone https://github.com/kubernetes/kubernetes.git
git clone https://github.com/bxy09/k8s-debian.git cluster/debian
```

## 预先步骤
1. 确保所有节点都安装了新版本的Docker,Sudo,bridge-utils。（实测，bridge-utils没安装的话，也可以通过人工方式将flannel网络加入到路由表中）。
2. 节点可以互相通信，为了方便起见，都设定好免密登录。
3. 每个节点都可以访问到docker repository，如果不能访问互联网，请设定自己内部可以访问到的源，修改pause以及其他addon所需要的image的地址。
4. 确保节点是通过systemd进行服务配置。
5. 测试过的版本为 k8s-1.1.4, flannel-0.5.5 , etcd-2.2.1。其他版本可能因为启动配置参数的变化，需要进行一些调整，在util.sh中`create-*-opts`各个函数的启动参数设定进行修改。

## 下载执行文件
运行download-release.sh下载执行文件，可以通过修改以下环境变量修改下载的版本：
```
FLANNEL_VERSION
ETCD_VERSION
KUBE_VERSION
```
其中etcd用于保证高可用性与一致性的配置设定与共享；flannel用于建立vnets实现不同节点之间的pod间通信；Kube则是主角k8s。

下载完成后的可执行文件放于`binaries/master`与`binaries/minion`中，分别对应于master节点和minion节点所需要的可执行程序。

## 设定节点角色
修改config-default.sh
```
nodes 节点名称
num_nodes 节点数目
role 各个节点的角色，a为master，i为minion
service_cluster_ip_range 是系统内用于定义服务以及实现负载均衡的虚拟ip段
flannel_net 是flannel用来定义pod通信网络的
```
请注意，如果修改了`service_cluster_ip_range`,请也相应修改之后的`DNS_SERVER_IP`，这个IP定义了DNS服务对应的虚拟IP。

该版本的配置只支持单master节点的集群，之后将研究多master节点从而实现High Availability。

## 布置集群
直接启动脚本，完成配置。
```
KUBERNETES_PROVIDER=debian ./cluster/kube-up.sh
```
以下简要介绍以下实际工作步骤，以及debian版本所做的修改。

### kube-up
./cluster/kube-up.sh 中启动 ./cluster/debian/util.sh 中的kube-up函数。
首先加载之前填入的config-default.sh中的配置。

setClusterInfo从配置中nodes与roles得到得到master节点Master_IP和minion众节点Node_IPS。

其后则是遍历各个节点由各个节点不同角色启动不同的配置函数。

#### provision-masterandnode
以该节点同时担任master节点和minion节点为例，其他情况为该情况的子集。

首先在目标节点的用户目录下建一个临时的文件夹`~/kube`，并且将可执行程序与配置文件都拷贝到节点。

EXTRA_SANS是附属在之后建立的签名中的，详情参考[k8s Authentication Plugins](http://kubernetes.io/v1.1/docs/admin/authentication.html)。

之后主要是生成各个组件的默认配置，包括etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, kube-proxy, , kube-proxy 以及 flanneld。其中kubelet, kube-proxy 是起到minion作用的，flannel是所有节点都要的，剩下的都是master所必须的。

其中，为了适应中国的网络环境，在kubelet的配置中添加了pod-infra-container-image=daocloud.io/gpx_dev/pause:0.8.0，pause image的作用在于启动一个什么都不干的container作为pod内其他container共享文件系统、网络等namespace的公共区域。如果不加入这个设置，则系统将会向Google自己的docker源拖取镜像。

然后则用管理员权限，将默认配置复制到`/etc/default`目录下，将systemd配置复制到`/lib/systemd/system`目录下。
生成签名证书到`/svc/kubernetes`目录下。
复制可执行程序到`/opt/bin`目录下。
启动etcd服务。

reconfDocker中，一方面在etcd中登记flanneld网络的配置，另一方面修改docker systemd配置，以及`/etc/default/docker`中的默认配置，使得Docker启动的container改用flanneld网络。

#### 后续
`verify-cluster`通过远程登录，确认各个节点都启动了相应的模块。
`detect-master` 拿到KUBE_MASTER_IP。之后一系列`kubectl`命令都是指向该节点的api服务器。

`load-or-gen-kube-basicauth` 和 `kube-config` 进行一些账户与安全的设置等。

## 安装附件
重要的附件包括配置DNS，Kube-UI, KFE(Kibana Fluentd Elasticsearch)。将在下一篇博文中进行介绍。
