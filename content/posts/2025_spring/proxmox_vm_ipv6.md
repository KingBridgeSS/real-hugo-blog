---
title: "Proxmox 虚拟机和容器化独立 IPV6 设置"
date: 2025-04-08
tags : [
    "misc",
]
type: "post"
showTableOfContents: true
---

环境：PVE和虚拟机都可以申请独立的IPV6。假设IPV6网关为：

`1111:2222:3333:4444::1`

可申请的IPV6段为：

`1111:2222:3333:4444::/64`

# PVE配置
这里vmbr0直通物理网卡，vmbr1用于PVE内的NAT。IPV6走vmbr0。

假设给PVE的IPV6申请了

`1111:2222:3333:4444::2/64`

vi /etc/network/interfaces

```go
root@pve:~# cat /etc/network/interfaces
auto lo
iface lo inet loopback

iface enp1s0 inet manual

iface wlp2s0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 11.22.33.2/24  
        gateway 11.22.33.1
        bridge-ports enp1s0
        bridge-stp off
        bridge-fd 0
iface vmbr0 inet6 static
        address 1111:2222:3333:4444::2/64
        gateway 1111:2222:3333:4444::1
auto vmbr1
iface vmbr1 inet static
        address 192.168.233.10/24
        #gateway 192.168.233.1
        bridge_ports none
        bridge_stp off
        bridge_fd 0
```

应用配置

```plain
systemctl restart networking
```

可以通过

```plain
curl 6.ipw.cn
```

验证IPV6是否开启。



接下来编辑 

vi /etc/sysctl.conf

确保这两行的开启

```plain
net.ipv6.conf.all.forwarding=1                                                                                                                                                                                   
net.ipv6.conf.all.proxy_ndp=1
```

执行 sysctl -p 生效。



接下来配置ndp代理。使用 ndppd 比较方便。

`apt install ndppd`

接下来 vi /etc/ndppd.conf

```go
route-ttl 30000
proxy vmbr0 {
    rule 1111:2222:3333:4444::/64 {
        auto
    }
}
```

启动服务：systemctl enable --now ndppd



# 虚拟机配置
虚拟机使用上面的vmbr0网桥，然后手动配置IPV6即可，例如ubuntu的netplan。

如果要为docker container使用IPV6，可以在docker启动的时候添加参数`--network host`直接使用HOST的IPV6。 

