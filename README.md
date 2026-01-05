# openwrt-vm docker 

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