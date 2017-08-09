---
title: Kubernetes的一些基本插件，并且在墙内进行布置
date: 2016-02-16 16:11:49
tags: kubernetes, docker, 云
---

继上一篇博文之后，再介绍一些必要的K8s插件，并且将镜像搬入到国内。
介绍的插件包括：
1. DNS Kubernetes内部的DNS，方便各个服务进行内部的服务发现。
2. fluentd-elasticsearch/KFE (Kibana, Fluentd, Elasticsearch) 日志收集与展示。
3. Dashboard web界面。

Web UI现在官方有两个方案，但都没有达到稳定版本，因此不在此介绍。

代码参见https://github.com/bxy09/k8s-addons

基本参照自kubernetes/cluster/addons

<!--more-->
## DNS
相对于官方版本，只是将镜像从gcr搬运到daocloud.io


## fluentd-elasticsearch
相对于官方版本，将镜像搬运到国内，并且缩减fluentd配置中关于docker,
k8s的日志项，因为systemd配置下的日志并不会存于/var/log/下单独的文件中，
而是通过journald来进行管理。
如果需要记录这些日志，请通过专门的fluentd-journal 插件进行读取，
或者通过journalctl与nc的组合，将日志转发到网络接口。

另外，fluentd在每个节点上的运行时通过k8s的DaemonSet来进行的。
有别于一般常用的ReplicationController可以配置固定数量的pod，DaemonSet保证每个节点上运行一个pod。
DaemonSet目前需要v1beta版接口，因此需要在kube-apiserver中新加入参数(`--runtime-config=extensions/v1beta1/daemonsets=true`)并重启，添加该接口的支持。并且，还需要重启kube-controller-manager。

访问地址：http://master-ip:8080/api/v1/proxy/namespaces/kube-system/services/kibana-logging/

## Dashboard
相对于官方版本，将镜像搬运到国内daocloud.io。

访问地址：http://master-ip:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/
