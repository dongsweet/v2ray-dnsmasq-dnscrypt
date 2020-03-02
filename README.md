# V2Ray + Dnsmasq + DoH 上网 Step-by-step

本方案记录在OpenWRT路由器上搭建V2Ray客户端，实现网内自动使用该网络的步骤。
本文主要借鉴了felix-fly的[文章](https://github.com/felix-fly/v2ray-openwrt/blob/master/README.md)，也可见[本地README_ORIGINAL.md](README_ORIGINAL.md)，表示感谢！

## 主要原理

基于IPSET（名称：gw）实现V2Ray流量转发，使用DNSMASQ-FULL针对列表（园外）中的域名进行解析，解析该部分域名使用DoH（DNS over HTTPS），并将解析出的地址加入gw IPSET，实现该部分域名通过V2Ray出园。
主要优势：
* 名单以外的域名使用路由器原有DNS（ISP提供）解析
* 名单以内的域名使用DoH解析，防止污染
* DoH使用国内某专业服务，速度飞快
* DoH解析过后的地址自动使用V2Ray实现流量转发

## 升级DNSMASQ-FULL

若已完成升级请略过该步骤。

为避免自带DNSMASQ卸载后无法解析域名，需同时执行删除DNSMASQ和安装DNSMASQ-FULL。
```shell
opkg update
opkg remove dnsmasq && opkg install dnsmasq-full
```

## 安装V2Ray

可以从felix-fly的repo[release](https://github.com/felix-fly/v2ray-openwrt/releases)下找自己对应平台的文件，压缩包内只包含v2ray单文件，如果不喜欢可以自行从官方渠道下载。

创建v2ray目录：
```shell
mkdir -p /etc/config/v2ray
```

将v2ray主程序和配置文件（除主程序外，全部位于v2ray文件夹中）复制到/etc/config/v2ray目录：
```shell
v2ray           # v2ray主程序
v2ray.service   # v2ray服务，将创建ipset和iptables命令集成在该文件中
gw.hosts        # 一些园外网站列表，用于告知dnsmasq使用单独的
ad.hosts        # 一些广告网站列表，用于告知dnsmasq直接免解析
gw.ips          # 一些园外IP地址（段），用于配置ipset走V2Ray
ad.ips          # 一些广告IP地址，用于配置ipset直接拒绝
client.json     # v2ray配置文件，填入自己的服务器信息
```

关于client.json，inbound部分主要监听12345端口，使用任意门（dokodemo-door）协议；outbound部分则可参考V2RayN软件导出的配置文件。

创建服务，直接链接到/etc/init.d即可，添加可执行权限。
```shell
ln -s /etc/config/v2ray/v2ray.service /etc/init.d/v2ray
chmod +x /etc/init.d/v2ray
/etc/init.d/v2ray enable
```

## 安装DoH客户端

DoH客户端使用https-dns-proxy软件，可使用如下命令进行安装：
```shell
opkg install https-dns-proxy
```
安装完成后，建议使用此处提供的配置文件对自动下载的配置文件和init.d进行替换，需替换的文件如下：
```shell
/etc/init.d/https-dns-proxy    # 启动脚本，使用https-dns-porxy/init.d/https-dns-proxy替换
/etc/config/https-dns-proxy    # 配置文件，使用https-dns-porxy/config/https-dns-proxy替换
```
替换启动脚本的目的是取消启动时自动更新dnsmasq DNS转发字段功能。

替换配置文件的目的是将原有的Google、CloudFlare服务更换为可用的Rubyfish（国内）服务。


## 配置DNSMASQ

DNSMASQ默认使用/etc/dnsmasq.d目录作为扩充配置目录，该目录可能不存在，创建即可。
创建完成后，为gw.hosts、ad.hosts在该目录下创建符号链即可。
```shell
mkdir -p /etc/dnsmasq.d
ln -s /etc/config/v2ray/gw.hosts /etc/dnsmasq.d/gw.hosts
ln -s /etc/config/v2ray/ad.hosts /etc/dnsmasq.d/ad.hosts
```

## 启动服务

正常情况下，重启一次路由器即完成自动出园操作。
不想重启的话，可以单独重启一些服务。
```shell
/etc/init.d/v2ray restart
/etc/init.d/https-dns-proxy restart
/etc/init.d/dnsmasq restart
```