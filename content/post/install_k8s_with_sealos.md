---
title: "Install k8s with sealos"
date: 2021-11-08T20:10:23+08:00
draft: false

description: Tcpreplay使用Netmap模式的步骤
author: realzhangm
tags: ["记录", "k8s"]
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
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
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
  

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL
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
