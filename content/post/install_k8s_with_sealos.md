---
title: "Install k8s with sealos"
date: 2021-11-08T20:10:23+08:00
draft: false

description: Tcpreplay使用Netmap模式的步骤
author: realzhangm
tags: ["记录", "k8s", "安装"]
---

## 说明
本文记录了使用 sealos 安装 k8s 的过程，其中虚拟机使用 vagrant 来配置。

## 安装 vagrant
可以参考 [vagrant](https://www.vagrantup.com/downloads) 官网。
``` bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install vagrant
```
入门参考，[点这里](https://zhuanlan.zhihu.com/p/259833884)

## 使用 vagrant 启动虚拟机
以下为 Vagrantfile 文件，在这个文件的相同目录下可以执行 vagrant 命令。如，启动虚拟机 `vagrant up` 。
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64root"
  config.ssh.private_key_path = "/home/ingm/.ssh/id_rsa"

  config.vm.define "s0" do |s0|
    s0.vm.hostname = "s0"
    s0.vm.network "private_network", ip: "192.168.33.10"
    s0.ssh.username = "root"
    s0.ssh.password = "12345678"
    s0.ssh.insert_key = "false"
  end


  config.vm.define "s1" do |s1|
    s1.vm.hostname = "s1"
    s1.vm.network "private_network", ip: "192.168.33.11"
    s1.ssh.username = "root"
    s1.ssh.password = "12345678"
    s1.ssh.insert_key = "false"
  end

  config.vm.define "s2" do |s2|
    s2.vm.hostname = "s2"
    s2.vm.network "private_network", ip: "192.168.33.12"
    s2.ssh.username = "root"
    s2.ssh.password = "12345678"
    s2.ssh.insert_key = "false"
  end

  config.vm.define "s3" do |s3|
    s3.vm.hostname = "s3"
    s3.vm.network "private_network", ip: "192.168.33.13"
    s3.ssh.username = "root"
    s3.ssh.password = "12345678"
    s3.ssh.insert_key = "false"
  end


   config.vm.provider "virtualbox" do |vb|
     # Customize the amount of memory on the VM:
     vb.memory = "3000"
     vb.cpus = 2
   end
  
end
```

## 安装 sealos
进入到 s0 (192.168.33.10) 中。
下载 [sealos](https://github.com/fanux/sealos)，和 k8s 部署包。
```bash
vagrant ssh s0
# 下载并安装sealos, sealos是个golang的二进制工具，直接下载拷贝到bin目录即可, release页面也可下载
wget -c https://sealyun-home.oss-cn-beijing.aliyuncs.com/sealos/latest/sealos && \
    chmod +x sealos && mv sealos /usr/bin

# 下载离线资源包
wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/05a3db657821277f5f3b92d834bbaf98-v1.22.0/kube1.22.0.tar.gz

```

执行如下安装命令：
```bash
sealos init \
--master 192.168.33.10 \
--node 192.168.33.11 \
--node 192.168.33.12 \
--user root \
--passwd '12345678' \
--interface enp0s8 \
--pkg-url kube1.22.0.tar.gz \
--version v1.22.0
```

出现以下图标后，检查安装是否成功：
![ok](/img/sealos_ok.png)

```bash
kubectl get node -owide
kubectl get pod --all-namespaces
```

`kubectl get node -owide` 是发现所有的 node ip 全是 enp0s3 网卡接口的 IP， 应该是 kubelet 默认使用了第一个网卡的 IP 地址。
会有照成一些问题，需要修改一些。登录到每台虚拟中，修改 kubelet 的启动参数，设置 node-ip 为 192.168.33 网段的 IP。
[参考文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#workflow-when-using-kubeadm-init)

```bash
# 加入 KUBELET_KUBEADM_ARGS="--node-ip=192.168.33.11 ...原有配置"
vim /var/lib/kubelet/kubeadm-flags.env

# 重启 kubelet
systemctl daemon-reload && systemctl restart kubelet
```


## 示例 1 (安装单机版本 redis)

`redis-standalone.yml` 文件如下，使用是执行： `kubectl apply -f redis-standalone.yml`

```yaml
# 定义配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-standalone-conf
data:
  redis.conf: |
        bind 0.0.0.0
        port 6379
        requirepass 111111
        appendonly yes
        cluster-config-file nodes-6379.conf
        pidfile /redis/log/redis-6379.pid
        cluster-config-file /redis/conf/redis.conf
        dir /redis/data/
        logfile /redis/log/redis-6379.log
        cluster-node-timeout 5000
        protected-mode no
---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-standalone
spec:
  replicas: 1
  serviceName: redis-standalone
  selector:
    matchLabels:
      name: redis-standalone
  template:
    metadata:
      labels:
        name: redis-standalone
    spec:
      initContainers:
      - name: init-redis-standalone
        image: busybox
        command: ['sh', '-c', 'mkdir -p /redis/log/;mkdir -p /redis/conf/;mkdir -p /redis/data/']
        volumeMounts:
        - name: data
          mountPath: /redis/
      containers:
      - name: redis-standalone
        image: redis:5.0.6
        imagePullPolicy: IfNotPresent
        command:
        - sh
        - -c
        - "exec redis-server /redis/conf/redis.conf"
        ports:
        - containerPort: 6379
          name: redis
          protocol: TCP
        volumeMounts:
        - name: redis-config
          mountPath: /redis/conf/
        - name: data
          mountPath: /redis/
      volumes:
      - name: redis-config
        configMap:
          name: redis-standalone-conf
      - name: data
        hostPath:
          path: /redis/
---
# NodePort 服务，可以通过集群的任何节点访问，包括主节点
kind: Service
apiVersion: v1
metadata:
  labels:
    name: redis-standalone
  name: redis-standalone
spec:
  type: NodePort
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
    nodePort: 31379
  selector:
    name: redis-standalone
```
查看创建的资源
```
kubectl get statefulset
kubectl get pod
kubectl get svc
```

登录到 pod 中
```
kubectl exec -it redis-standalone-0 -- /bin/bash
```

使用 redis 客户端测试：
```shell
redis-cli -h 192.168.33.10 -p 31379 -a 111111
redis-cli -h 192.168.33.11 -p 31379 -a 111111
redis-cli -h 192.168.33.12 -p 31379 -a 111111
```
---