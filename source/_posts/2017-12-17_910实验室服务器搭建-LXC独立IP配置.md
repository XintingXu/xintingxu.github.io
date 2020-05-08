title: 910实验室服务器搭建-LXC独立IP配置
date: 2017-12-17 19:39:37
tags: [实验室服务器搭建]
categories: [log]
---
LXC容器的默认IP，是通过网桥"lxcbr0"分配的10.0.3.XXX的局域网IP，除了host主机外，其他用户无法访问。

## 解决方案
可以通过端口映射，或者更换网桥的方法，将容器和host置于同一个网段，使得外部用户也可以登入容器内部。

因为我们涉及的用户比较多，采用端口映射的方法不仅麻烦，还给使用人员带来不便。

<!-- more -->

# 独立IP设置
## 添加网桥
通过ifconfig，查看当前的网络信息，不同的环境有不同的命名，但一般有一个外网接口enXX、一个本地回环l0、一个LXC容器的默认网桥lxcbr0。

我们需要实现的，是创建一个网桥，使得LXC容器能够分享本机网卡。

修改"/etc/networking/interfaces"
添加
```
auto br0
iface br0 inet dhcp
	bridge-ifaces XXX
	bridge-ports XXX
	up ifconfig XXX up

iface XXX inet manual
```
其中"XXX"即为原有的网络接口名称。

修改完毕后，重启系统。

如果没有问题的话，可以再次通过ifconfig查看当前的网络状况，网桥br0应该配好了。

## 修改LXC配置文件
修改"/etc/lxcf/default.conf"，以及现有容器的配置文件。

将原有的项目修改为：
```
lxc.network.link = br0
```

其中"br0"即为上一步创建的网桥名称，如果命名为其他，此处也应该对应修改。

## 检查结果
以上操作完成后，在容器内查看到的IP地址，应该就是和主机同网段的IP。这样，宿主机所在网络的其他用户，也可以访问容器了。