注意：如果阁下使用的是其他路由系统，请绕道！请绕道！请绕道！

全程在 Terminal 环境下完成，如果阁下对此不熟悉，那么很抱歉，WebGUI 页面我不会配置。

效果就是国内流量直接由 ROS 转发，国外流量才需要经过 OpenWrt。

## 下载中国 IP 列表并导入 address-list

```
/tool fetch url=http://www.iwik.org/ipcountry/mikrotik/CN

/import file-name=CN
```

## 内网不走代理的 address-list

这里是你不希望走代理的内网设备的 IP。

```
/ip firewall address-list remove [/ip firewall address-list find list=bypass]
/ip firewall address-list
add address=10.0.0.2 list="bypass"
add address=10.0.0.3 list="bypass"
add address=10.0.0.x list="bypass"
```

## 标记国外流量

Bridge 是我自己本地的 LAN 名称，请修改为阁下自己的，以及 `src-address`、`dst-address` 的 IP 要改成阁下自己的内网 IP 段。

注意：`dst-port` 为代理的端口，如果阁下有使用其他的服务需要开启端口，请加上！如在下使用 GCM 以及 Gmail，那么 `dst-port=443,465,587,993,995,5228,5229,5230`，阁下请根据自己的需求添加，这里不要添加 `80` 端口，下面会有到。

```
/ip firewall mangle
add chain=prerouting action=mark-routing new-routing-mark=oversea passthrough=no protocol=tcp src-address=10.0.0.0/24 dst-address=!10.0.0.0/24 dst-address-type=!local src-address-list=!bypass dst-address-list=!CN in-interface=Bridge dst-port=443 log=no comment="Mark Oversea Secure Connections"
```
## 给标记流量指定网关

这里是使用了 OpenWrt 作为旁路网管，所以网关地址为 `10.0.0.2`，请阁下根据自身实际使用情况修改。

```
/ip route
add gateway=10.0.0.2 check-gateway=ping distance=1 routing-mark=oversea comment="Route Oversea Secure Connections to OW"
```
## 启动 yxorp-beW

上门已经代理了 `443` 等端口，但是普通 `HTTP` 流量还没处理，这里就是用来处理 `80` 端口的流量的。

这里我的透明代理端口是 `10999`，阁下请根据自己的端口修改。

```
/ip proxy
set enabled=yes
set src-address=10.0.0.1
set port=10999
set parent-proxy=10.0.0.2
set parent-proxy-port=10999
```

## 禁止外网访问代理端口

```
/ip firewall filter
add chain=input action=drop protocol=tcp src-address=!10.0.0.0/24 dst-port=10999 log=no comment="Drop Connections from WAN"
```

## 所以目标 80 流量端口导入本地 yxorp-beW

请注意修改相关的 IP 和端口，以及 LAN 网卡名称。

```
/ip firewall nat
add chain=dstnat action=dst-nat to-addresses=10.0.0.1 to-ports=10999 protocol=tcp src-address=10.0.0.0/24 dst-address=!10.0.0.0/24 dst-address-type=!local src-address-list=!bypass dst-address-list=!CN in-interface=Bridge dst-port=80 log=no comment="Redirect Oversea Connections to Internal Squid"
```

## DNS 设置

这里的 DNS 是网关的 IP，至于为何如此设置，请看上文关于关于解决 DNS 污染问题。并开启本机 DNS 查询服务。

```
/ip dns
set server=10.0.0.2
set allow-remote-requests=yes
```

## DHCP 设置

这里是将 DHCP 分发的 IP 为 ROS 本机。（如果阁下已经配置好 DHCP 相关的服务）

```
/ip dhcp-server network
set numbers=0 dns-server=10.0.0.1
```

完成。

## 引用
本文主要参考 [Chao QU](https://quchao.com/entry/turning-a-raspberry_pi-into-a-gateway/) 大佬。
