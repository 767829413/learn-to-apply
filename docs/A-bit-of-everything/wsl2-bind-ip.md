# 在win10的WSL2固定ip

## 固定WSL2的ip地址

### 前提条件

* [已经安装好WSL2](../Productivity/wsl2-dev.md)

### 解决方案

这个主要是借鉴老外提出的[建议](https://lifesaver.codes/answer/static-ip-on-wsl-2-418)

就是每次开机,去分配一个固定ip给wsl2和主机

```cmd
wsl -d Ubuntu -u root ip addr add 192.168.50.16/24 broadcast 192.168.50.255 dev eth0 label eth0:1
netsh interface ip add address "vEthernet (WSL)" 192.168.50.88 255.255.255.0
```
