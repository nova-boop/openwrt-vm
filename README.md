# openwrt-vm docker 


## 项目说明 [![](https://img.shields.io/badge/-项目基本介绍-FFFFFF.svg)](#项目说明-)
- 固件来源：[![Lean](https://img.shields.io/badge/Lede-Lean-ff69b4.svg?style=flat&logo=appveyor)](https://github.com/coolsnowwolf/lede) [![P3TERX](https://img.shields.io/badge/OpenWrt-P3TERX-blueviolet.svg?style=flat&logo=appveyor)](https://github.com/P3TERX/Actions-OpenWrt) [![Flippy](https://img.shields.io/badge/Package-Flippy-orange.svg?style=flat&logo=appveyor)](https://github.com/unifreq/openwrt_packit) [![demo](https://img.shields.io/badge/Build-nova_boop-32C955.svg?style=flat&logo=appveyor)](https://github.com/nova-boop/openwrt-vm)
- 项目使用 Github Actions 拉取 [Lean](https://github.com/coolsnowwolf/lede) 的 Openwrt 源码仓库进行云编译
- 固件默认管理地址：`192.168.2.3` 默认用户：`root` 默认密码：`password`
- 固件集成的所有 ipk 插件全部打包在 Packages 文件中，可以在 [Releases](https://github.com/nova-boop/openwrt-vm/releases) 内进行下载
- 项目编译的固件插件为最新版本，最新版插件可能有 BUG，如果之前使用稳定则无需追新
- 第一次使用请采用全新安装，避免出现升级失败以及其他一些可能的 BUG
- 插件过多的固件会产生较多的临时文件，请保证磁盘介质空间足够
  - VM版本    引导空间:256MB 固件空间 1024MB ，建议安装盘不少于 4GB
 

## **docker openwrt 旁路由配置指南 支持 公网 ipv6**

<font color=red> ⛔ **(注意 由于 使用的是 docker macvlan 模式，宿主机 将无法访问 到macvlan，也就是说 宿主机 无法访问到 容器)**</font>

### **宿主机网络相关配置**

```
# 开启网卡混杂模式(需要网线连接的网口，wifi无法使用混杂模式) 
# enp0s31f6 网卡名称(根据自身电脑网口名称来) ifconfig or ip addr
sudo ip link set enp1s0 promisc on

# docker 开启 ipv6 支持 ip6tables = true
vim /etc/docker/daemon.json

{
  "data-root": "/home/username/docker-root",
  "experimental": true,
  "ip6tables": true
}

# 重启 docker
sudo systemctl restart docker
```

### **创建 docker macvlan 虚拟网卡**

```
# subnet、gateway 根据 局域网环境来填写
# 此处的 ipv6 网关和 
docker network create -d macvlan \
--subnet=192.168.2.0/24 \
--gateway=192.168.2.1 \
--ipv6 \
--subnet=fddb:b748:79d::/64 \
--gateway=fddb:b748:79d::1 \
-o parent=enp1s0 macnet
```

### **部署 openwrt 容器**

```
# 导入 容器到 docker 镜像
docker import openwrt-x86-64-generic-rootfs.tar.gz openwrt:vm

# 启动 openwrt 容器
docker run -dit --name=vm \
--restart=always  \
--hostname=9847714a2b0a \
--ip=192.168.2.3 \
--network=macnet \
--privileged \
openwrt:vm /sbin/init
```

### **修改openwt 默认的 network 配置 支持 ipv6**

```
# 配置容器 lan
# ipaddr=运行容器时指定的ip  
# gateway=局域网路由器网关 
# dns=局域网dns解析(默认一般都是 gateway 地址) 
vim  /etc/config/network

config interface 'lan'
        option proto 'static'
        option ipaddr '192.168.2.3'
        option netmask '255.255.255.0'
        option gateway '192.168.2.1'
        option dns '192.168.2.1'
        option ip6assign '64'
        option device 'br-lan'

config interface 'LAN6'
        option proto 'dhcpv6'
        option reqaddress 'try'
        option reqprefix 'auto'
        option device '@lan'

config device
        option name 'br-lan'
        option type 'bridge'
        list ports 'eth0'
```

### **openwrt 在 docker 有些服务不会自启，或者自启失败，需要命令执行重启操作**

```
# 编辑 /etc/rc.local 将需要自启动的 服务 写入 自启脚本
vim /etc/rc.local
/usr/sbin/uhttpd -p 80 -h /www
/etc/init.d/pushbot restart
/etc/init.d/passxxxx restart
/etc/init.d/passxxxx_server restart
/etc/init.d/v2ray_server restart
exit 0
```

### **开启 ipv6 支持(编辑完之后重启容器)**

[**此部分教程需要感谢这篇求助帖 来自网络**](https://meta.appinn.net/t/topic/46007)

```
# 编辑 /etc/sysctl.conf
vim /etc/sysctl.conf
net.ipv6.conf.all.disable_ipv6=0
net.ipv6.conf.default.disable_ipv6=0
net.ipv6.conf.default.accept_ra=2
net.ipv6.conf.all.accept_ra=2
```

# 设置默认主题

```shell
uci set luci.main.mediaurlbase='/luci-static/argon'

uci set luci.main.mediaurlbase='/luci-static/bootstrap'

uci commit luci
```


## 特别提示 [![](https://img.shields.io/badge/-个人免责声明-FFFFFF.svg)](#特别提示-)

- **因精力有限不提供任何技术支持和教程等相关问题解答，不保证完全无 BUG！**

- **本人不对任何人因使用本固件所遭受的任何理论或实际的损失承担责任！**

- **本固件禁止用于任何商业用途，请务必严格遵守国家互联网使用相关法律规定！**

- **请务必遵从 “不涉及政治，不涉及宗教，不涉及黄赌毒” 三不原则！**


## 鸣谢 [![](https://img.shields.io/badge/-跪谢各大佬-FFFFFF.svg)](#鸣谢-)
| [ImmortalWrt](https://github.com/immortalwrt) | [coolsnowwolf](https://github.com/coolsnowwolf) | [P3TERX](https://github.com/P3TERX) | [Flippy](https://github.com/unifreq) | [haiibo](https://github.com/haiibo) |
| :-------------: | :-------------: | :-------------: | :-------------: | :-------------: |
| <img width="100" src="https://avatars.githubusercontent.com/u/53193414"/> | <img width="100" src="https://avatars.githubusercontent.com/u/31687149"/> | <img width="100" src="https://avatars.githubusercontent.com/u/25927179"/> | <img width="100" src="https://avatars.githubusercontent.com/u/39355261"/> | <img width="100" src="https://avatars.githubusercontent.com/u/85640068"/> |
| [Ophub](https://github.com/ophub) | [SuLingGG](https://github.com/SuLingGG) | [QiuSimons](https://github.com/QiuSimons) | [IvanSolis1989](https://github.com/IvanSolis1989) |
| <img width="100" src="https://avatars.githubusercontent.com/u/68696949"/> | <img width="100" src="https://avatars.githubusercontent.com/u/22287562"/> | <img width="100" src="https://avatars.githubusercontent.com/u/45143996"/> | <img width="100" src="https://avatars.githubusercontent.com/u/44228691"/> | |


<a href="#readme">
<img src="https://img.shields.io/badge/-返回顶部-FFFFFF.svg" title="返回顶部" align="right"/>
</a>
