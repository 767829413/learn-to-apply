# ubuntu24.04 openvpn踩坑指南

最近笔记本越来越烫手，虽然也是16G的内存，但是一开六七个vscode窗口就打不开docker了，一气之下就怒转ubuntu。正好闲置了一台nuc,直接开搞，具体过程不表，这里只讲openvpn使用过程中各种问题

## 问题1：踏马的ubuntu环境的dns解析需要自己配置脚本

```ovpn
# 配置文件末尾请加上这个
script-security 2
up /etc/openvpn/update-systemd-resolved
down /etc/openvpn/update-systemd-resolved
```

具体原因请看： <https://serverfault.com/questions/732317/openvpn-and-systemd-resolved>

## 问题2：用了docker？嘿嘿，小心ip解析冲突

```bash
ip route 
default via 192.168.20.254 dev eno1 proto dhcp src 192.168.20.44 metric 20100 
10.0.255.0/24 dev tap0 proto kernel scope link src 10.0.255.25 
10.20.0.0/16 via 10.0.255.1 dev tap0 
10.21.0.0/20 via 10.0.255.1 dev tap0 
100.103.0.0/16 via 10.0.255.1 dev tap0 
172.16.0.0/12 via 10.0.255.1 dev tap0
## SB解析出来内网的ip正好是 172.17.2.9 这不巧了不是
172.17.2.0/24 via 10.0.255.1 dev docker0 
192.168.20.0/24 dev eno1 proto kernel scope link src 192.168.20.44 metric 100 
```

解决方案及其简单,

```bash
sudo iptables -t nat -A POSTROUTING -s 172.17.2.0/24 -o tap0 -j MASQUERADE
```
