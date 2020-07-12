---
title: hostapd + dhcpd + nftables 在 CentOS 7 上开 WLAN 热点
date: 2017-03-03 06:12:04
categories:
- '应用实践'
tags:
- 'Linux'
---
因为 NetworkManager 的开热点功能太废柴，所以只好另辟蹊径。

首先介绍一下这三款软件

- **hostapd** : 至今为止用得最广泛的无线热点程序，稳定而强大，几乎你能想到的无线路由器都在使用它。
- **dhcpd** : 强大的 DHCP 服务器（动态主机服务），适合用于管理多个大型网络的主机地址自动分配。
- **nftables** : 新兴的一个网络过滤器。因为业界稳定使用 15 年之久的 iptables 的利于新手上手，而且 CentOS 7 后， iptables 不再是一个独立的系统服务，而是包了一层叫 firewalld 的壳，它会托管 iptables 并往里面塞很多杂乱的默认规则，导致比较难下手。所以这里使用 nftables 代替传统路由器使用的 nftables。

开热点需要有能够支持 AP 模式的无线网卡，如果你不确定的话可以这样查看

	iw list
	
如果里面 **Supported interface modes:** 有　"AP" 那么意味着你的网卡支持。

首先需要安装这三个软件。在安装之前，请确保你的 CentOS 已经使用了 epel 源：

	yum install epel-release

然后开始安装它们

	yum install hostapd dhcpd nftables
	
dhcpd 在 CentOS 里一般自带。

安装完成之后，就是配置了。这三个软件都是作为服务启动，所以可以直接通过 systemctl 管理它们的自启动以及开关。

	systemctl start|stop|enable|disable hostapd|dhcpd|nftables

接下来的步骤需要多次提权访问，为了方便请直接 `sudo su` 后进行操作。

首先我们需要配置 hostapd ，它的默认配置文件在 `/etc/hostapd/hostapd.conf` 。这里只讲最基本的配置，其他更多的详细高级配置请参考本文最后的连接。

	#hostapd 提供一个控制终端 hostapd_cli，这里配置终端接入的细节
	ctrl_interface=/var/run/hostapd
	ctrl_interface_group=whell
	
	#一些比较普通的配置。。。。
	macaddr_ac1=0
	#这里配置使用哪一种认证方式，0 就是开放， 1 使用 WPA 系列，2 为任意，建议选 1
	auth_algs=1
	ignore_boardcast_ssid=0
	
	#下面配置 WPA 和 WPA2 的选项
	wpa=3
	wpa_key_mgmt=WPA-PSK
	wpa_pairwise=TKIP
	rsn_pairwise=CCMP
	
	#这里配置你的 WI-FI 密码
	wpa_passphrase=SomePassword
	
	#配置驱动，这里一般不需要更改
	driver=nl80211
	
	#配置 WI-FI 硬件，这里改成你 WI-FI 硬件的设备名称，可以用 nmcli device show 或者 ifconfig 查找
	interface=wlp2s0
	#这里配置频率模式
	hw_mode=g
	#频道
	channel=7
	#SSID，WI-FI 名称
	ssid=SSID_NAME
	#是否用 UIF-8 编码 WI-FI 名称，启用的话就能使用各种字符来命名 WI-FI 了，但在 Windows 上面普遍会出现 WI-FI 名称乱码问题
	utf8-ssid=1


保存之后别急着启动，因为默认情况下网卡受 NetworkManager 托管，所以 hostapd 无法管理无线网卡。执行下面命令以让 NetworkManager 不再托管无线网卡。

	nmcli device set wlp2s0 managed no

然后才启动 hostapd 服务，没有错误就证明配置好了。

接下来先给启动了的无线网卡配置 IP 地址

	ip addr add 192.168.2.1/24 dev wlp2s0

这个地址相当于平时配路由器的路由器地址，可以根据你的需要改成其他的，不过注意下面的所有配置都得跟着改动。

*后面的 `/24` 是 `255.255.255.0` 的缩写，如果想改动请自行查询子网掩码的格式

dhcpd 的配置文件默认在 `/etc/dhcp/dhcpd.conf`，这个文件夹需要 root 权限才能进入，建议执行 `sudo su` 。

	#让 DDNS 自动刷新，适合用在这种经常变动的网络中
	ddns-update-style interim;

	#监听 192.168.2.0 这个子网
	subnet 192.168.2.0 netmask 255.255.255.0 {
	
	        #设置自动派 IP 的范围
	        range 192.168.2.10 192.168.2.250;
	
	        #默认的 DNS 服务器
	        option domain-name-servers 223.6.6.6,223.5.5.5;
	        #告诉客户端路由器地址
	        option routers 192.168.2.1;
	        #告诉客户端子网掩码
	        option subnet-mask 255.255.255.0;
	        #告诉客户端广播地址
	        option broadcast-address 192.168.2.255;
	
	        #IP 默认租期和最大租期
	        default-lease-time 86400;
	        max-lease-time 172800;
	
	}

保存之后启动 dhcpd 服务，启动没有问题就可以了。

接下来就是 nftables 的配置了。nftables 不需要什么配置文件，只要打命令就行了。

	nft add table nat
	nft add chain nat post { type nat hook postrouting priority 0 \; }
	nft add chain nat pre { type nat hook prerouting priority 0 \; }
	
	#使得子网内的流量都通过 enp3s0 出去，这个是指出口的设备，也可以通过上面的命令查出来
	nft add rule nat post ip saddr 192.168.2.0/24 oif enp3s0 snat 192.168.2.1
	
到此你的电脑就能变成无线热点了。
	
##参考

- [Configuring a DHCP Server](https://www.centos.org/docs/5/html/Deployment_Guide-en-US/s1-dhcp-configuring-server.html)
- [About hostapd](https://wireless.wiki.kernel.org/en/users/documentation/hostapd)
- [将DEBIAN配置为软路由](https://blog.chou.it/2014/03/make-debian-a-home-router/)
- [Linux DHCP配置的完美攻略](http://network.51cto.com/art/201008/222644_all.htm)
- [Software Access Poiont](https://wiki.archlinux.org/index.php/Software_access_point_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
- [Wireless network configuration](https://wiki.archlinux.org/index.php/Wireless_network_configuration#Respecting_the_regulatory_domain)
- [nftables - Arch Wiki](https://wiki.archlinux.org/index.php/nftables)[](``)