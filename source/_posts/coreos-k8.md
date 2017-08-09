---
title: CoreOS 上布置K8节点
date: 2016-11-12 21:55:22
tags:
---

# CoreOS 介绍

# 布置方案
CoreOS 系统在启动时，会加载系统配置文件([Cloud-config](https://coreos.com/os/docs/latest/cloud-config.html))。
该配置文件的路径由系统安装时决定，青云所配置的路径为：`/usr/share/oem/cloud-config.yml`

其中的配置的编写方式，参考 CoreOS 系统[官方文档](https://coreos.com/os/docs/latest/cloud-config.html)进行配置。

我们所采用的 kube-slave 配置方案如下：
```yaml
#cloud-config
#/usr/share/oem/cloud-config.yml
write_files:
  # 扫描flannel环境文件是否就绪，一共扫描五次，每次间隔1秒
  - path: "/opt/bin/check-flanneld-file"
    permissions: "0755"
    owner: "root"
    content: |
      #!/bin/bash
      for i in {1..5}; do
      if [[ -f /run/flannel_docker_opts.env ]]; then
        exit 0
      fi
      sleep 1
      done
      exit 1
  - path: "/opt/bin/addr"
    permissions: "0755"
    owner: "root"
    content: |
      #!/bin/bash
      #/opt/sbin/addr
      ip addr show dev eth0 scope global|awk '/inet/{print substr($2,0,length($2)-3)}'
coreos:
  # 配置flannel参数，可以考虑不使用默认的flannel配置，因为默认配置采用从coreos拖取的镜像，在版本更新时，启动过慢
  flannel:
    etcd_endpoints: "http://192.168.100.100:4001"
  # docker配置加入启动前对 flannel 配置文件的确认
  units:
    - name: "docker.service"
      command: "start"
      drop-ins:
        - name: "exec_pre.conf"
          content: |
            [Service]
            ExecStartPre=/opt/bin/check-flanneld-file
    # kubelet 配置
    - name: "kubelet.service"
      command: "start"
      enable: true
      content: |
        [Unit]
        Description=Kubelet service
        After=flanneld.service

        [Service]
        Environment='KUBELET_OPTS=\
          --api-servers=http://192.168.100.100:8080 \
          --logtostderr=true \
          --cluster-dns=192.168.3.10 \
          --cluster-domain=cluster.local \
          --pod-infra-container-image=daocloud.io/gpx_dev/pause:0.8.0 \
          --config='
        ExecStart=/usr/bin/env bash -c '/opt/bin/kubelet --hostname-override=`/opt/bin/addr` $KUBELET_OPTS'
        Restart=always
        RestartSec=5

        [Install]
        WantedBy=multi-user.target
    # kube-proxy 配置
    - name: "kube-proxy.service"
      command: "start"
      enable: true
      content: |
        [Unit]
        Description=Kube-proxy service
        After=flanneld.service

        [Service]
        Environment='KUBE_PROXY_OPTS=\
          --master=http://192.168.100.100:8080 \
          --logtostderr=true'
        ExecStart=/usr/bin/env bash -c '/opt/bin/kube-proxy --hostname-override=`/opt/bin/addr` $KUBE_PROXY_OPTS'
        Restart=always
        RestartSec=5

        [Install]
        WantedBy=multi-user.target
    # 可能的文件挂载需求
    - name : "mnt-csvware.mount"
      command: "start"
      enable: true
      content: |
        [Unit]
        Description=CSV Data

        [Mount]
        What=/dev/sdb
        Where=/mnt/csvware
        Type=ext4
        Options=rw,relatime,errors=remount-ro,data=ordered

        [Install]
        WantedBy = multi-user.target
```
