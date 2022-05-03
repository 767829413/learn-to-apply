# 在win10的WSL2固定ip

## 固定WSL2的ip地址

### 前提条件

* [已经安装好WSL2](../../docs/Productivity/wsl2-dev.md)

### 解决方案

这个主要是借鉴老外提出的[建议](https://lifesaver.codes/answer/static-ip-on-wsl-2-418)

就是每次开机,去分配一个固定ip给wsl2和主机

首先获取本机的wsl列表,执行以下命令

```bash
wsl -l
```

输出大概是这样的

```text
PS your_path> wsl -l
适用于 Linux 的 Windows 子系统分发版:
docker-desktop-data (默认)
docker-desktop
Your_Ubuntu
```

然后就是执行下面的命令进行分配

```cmd
wsl -d Your_Ubuntu -u root ip addr add 192.168.50.16/24 broadcast 192.168.50.255 dev eth0 label eth0:1
netsh interface ip add address "vEthernet (WSL)" 192.168.50.88 255.255.255.0
```
