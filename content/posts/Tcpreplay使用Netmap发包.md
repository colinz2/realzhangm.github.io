---
title: "Tcpreplay使用Netmap发包"
description: "Tcpreplay使用Netmap模式的步骤"
date: "2018-01-14T09:28:42+08:00"
keywords: "发包, 报文回放"
---

# 说明
Tcprepaly是一个报文回放工具，可以将使用 .pcap 文件保存的报文回放。

Tcpreplay默认使用使用的标准的Linux系统API来发包的，发包速度较慢。Netmap是一种高效的收发报文的 I/O 框架，可以使用其API直接在用户态完成数据包到网卡的拷贝，即「内核旁路技术」[[^1]][[^2]]，从而大大提高发包效率。下面介绍了Tcpreplay使用Netmap的编译步骤。关于Tcprepaly和Netmap 的更是信息可以参考各自的官网。

# Netmap

1). 下载Netmap代码
```sh
git clone https://github.com/luigirizzo/netmap
```
2). 编译Netmap
```sh
./configure --drivers=i40e
make
```
3). 安装Netmap模块
```sh
# rmmod i40e
# insmod ./netmap.ko
# insmod ./i40e/i40e.ko

./configure --drivers=ixgbe
# rmmod ixgbe
# insmod ./netmap.ko
# insmod ./ixgbe/ixgbe.ko
```
## Tcpreplay

1). 下载代码
```sh
git clone https://github.com/appneta/tcpreplay
```
2). 编译安装
```sh
./configure --with-netmap=/home/zhangm/test/netmap/
make && make install
```
3). 使用

使用Tcpreplay时增加 --netmap 参数, 则使用 Netmap 模式

如:
```sh
tcpreplay -i ens1f0 -tK --loop 50000 --netmap /home/zhangm/pcap/bigFlows.pcap
```
-K, --preload-pcap Preloads packets into RAM before sending //提升效率

## 附:
##### 如何获取网卡驱动名称, 如 ens1f0 接口?
```sh
[root@localhost build]# ethtool -i ens1f0
driver: i40e
version: 2.3.6
firmware-version: 5.05 0x8000288a 0.0.0
expansion-rom-version: 
bus-info: 0000:02:00.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: yes
```
```sh
[root@localhost build]# lspci  | grep Eth
01:00.0 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
01:00.1 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
02:00.0 Ethernet controller: Intel Corporation Ethernet Controller X710 for 10GbE SFP+ (rev 02)
02:00.1 Ethernet controller: Intel Corporation Ethernet Controller X710 for 10GbE SFP+ (rev 02)
02:00.2 Ethernet controller: Intel Corporation Ethernet Controller X710 for 10GbE SFP+ (rev 02)
02:00.3 Ethernet controller: Intel Corporation Ethernet Controller X710 for 10GbE SFP+ (rev 02)
[root@localhost build]# lspci  -s 02:00.0 -vvv | grep driver
    Kernel driver in use: i40e
```


## 参考：
[^1]: 1:[https://blog.cloudflare.com/kernel-bypass/](https://blog.cloudflare.com/kernel-bypass/)
[^2]: 2:[http://blog.csdn.net/fengfengdiandia/article/details/52594758](http://blog.csdn.net/fengfengdiandia/article/details/52594758)
[^3]: 3:[http://blog.csdn.net/wwh578867817/article/details/49559453](http://blog.csdn.net/wwh578867817/article/details/49559453)
