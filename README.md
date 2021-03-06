# V2Ray + Dnsmasq + DoH 上网 Step-by-step

本方案记录在OpenWRT路由器上搭建V2Ray客户端，实现网内自动使用该网络的步骤。
本文主要借鉴了felix-fly的[文章](https://github.com/felix-fly/v2ray-openwrt/blob/master/README.md)，也可见[本地README_ORIGINAL.md](README_ORIGINAL.md)，表示感谢！

## 主要原理

### GW模式：

基于IPSET（名称：gw）实现V2Ray流量转发，使用DNSMASQ-FULL针对列表（园外）中的域名进行解析，解析该部分域名使用DoH（DNS over HTTPS），并将解析出的地址加入gw IPSET，实现该部分域名通过V2Ray出园。
主要优势：
* 名单以外的域名使用路由器原有DNS（ISP提供）解析
* 名单以内的域名使用DoH解析，防止污染
* DoH使用国内某专业服务，速度飞快
* DoH解析过后的地址自动使用V2Ray实现流量转发

### CN模式：

基于APNIC提供的中国IPv4列表进行过滤，除列表IP外，全部走V2Ray。同时，使用DNSMASQ-FULL针对列表（园外）中的域名进行解析，解析该部分域名使用DoH（DNS over HTTPS），防止污染。
主要优势：
* 名单以内的域名使用DoH解析，防止污染
* 更适合园内部分城市网络环境（访问多数园外网站都慢）

## 升级DNSMASQ-FULL

若已完成升级请略过该步骤。

为避免自带DNSMASQ卸载后无法解析域名，需同时执行删除DNSMASQ和安装DNSMASQ-FULL。
```shell
opkg update
opkg remove dnsmasq && opkg install dnsmasq-full
```

## 安装V2Ray

可以从felix-fly的repo [release](https://github.com/felix-fly/v2ray-openwrt/releases)下找自己对应平台的文件，压缩包内只包含v2ray单文件，如果不喜欢可以自行从官方渠道下载。
若felix-fly的repo无法找到合适的文件。可以试试kuoruan的repo [release](https://github.com/kuoruan/openwrt-v2ray/releases)，比如R2S设备，该repo下载的ipk文件，可以用opkg install进行安装。

创建v2ray目录：
```shell
mkdir -p /etc/config/v2ray
```

将v2ray主程序和配置文件（除主程序外，全部位于v2ray文件夹中）复制到/etc/config/v2ray目录。
为v2ray主程序创建链接
```shell
ln -s /etc/config/v2ray/v2ray /usr/bin/v2ray
```
使用kuoruan的ipk安装则无需复制主程序，也无需创建链接。
文件列表：
```shell
v2ray              # v2ray主程序（使用kuoruan则无）
v2ray-gw.service   # v2ray服务，使用GW模式
v2ray-cn.service   # v2ray服务，使用CN模式
doh.hosts          # 指定需要使用DoH进行解析的域名
gw.hosts           # GW模式下，指定流量需要走V2Ray的域名
ad.hosts           # 一些广告网站列表，用于告知dnsmasq直接免解析
gw.ips             # 一些园外IP地址（段），用于配置ipset走V2Ray
ad.ips             # 一些广告IP地址，用于配置ipset直接拒绝
client.json        # v2ray配置文件，填入自己的服务器信息
```

关于client.json，inbound部分主要监听12345端口，使用任意门（dokodemo-door）协议；outbound部分则可参考V2RayN软件导出的配置文件。

创建服务，直接链接到/etc/init.d即可，添加可执行权限。
```shell
ln -s /etc/config/v2ray/v2ray-xx(gw/cn).service /etc/init.d/v2ray
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
若添加重启DNSMASQ后，在日志中无法看到一长串类似于如下信息，则可能是DNSMASQ未使用/etc/dnsmasq.d目录作扩充。
```shell
 using nameserver 127.0.0.1#1053 for domain zzux.com
```
此时需在/etc/dnsmasq.conf中添加配置
```shell
conf-dir=/etc/dnsmasq.d
```
使用GW模式，则为doh.hosts和gw.hosts在/etc/dnsmasq.d下创建链接：
```shell
mkdir -p /etc/dnsmasq.d
ln -s /etc/config/v2ray/doh.hosts /etc/dnsmasq.d/gw.hosts
ln -s /etc/config/v2ray/gw.hosts /etc/dnsmasq.d/gw.hosts
```
使用CN模式，则为doh.hosts在/etc/dnsmasq.d下创建链接：
```shell
mkdir -p /etc/dnsmasq.d
ln -s /etc/config/v2ray/gw.hosts /etc/dnsmasq.d/gw.hosts
```

## 启动服务

CN模式下，第一次启动必须使用/etc/init.d/v2ray reload下载CN列表。
```shell
/etc/init.d/v2ray reload
/etc/init.d/https-dns-proxy restart
/etc/init.d/dnsmasq restart
```