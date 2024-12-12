# ubuntu 22.04 golang开发使用指瞎

## 开发工具-vscode

### debug设置

```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch Package",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "{your_code_path}",
            "asRoot": true,
            "console": "integratedTerminal",
            "env": {},
            "args": []
        }
    ]
}
```

使用了 `"asRoot": true`，需要保证 `sudo` 环境下的 `PATH` 变量包含 `Go` 的安装路径

```bash
## 编辑 /etc/sudoers 文件
sudo visudo

## 在文件中添加以下行（假设 Go 安装在 /usr/local/go/bin）
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin"

## 检查用户环境配置（如 ~/.bashrc 或 ~/.profile）中正确设置了 PATH 变量，并包含 Go 的路径
export PATH=$PATH:/usr/local/go/bin
```

## openvpn

### 配置文件

```ovpn
script-security 2
up /etc/openvpn/update-resolv-conf
down /etc/openvpn/update-resolv-conf
down-pre
```

***其实没啥用***

### dns设置（核心）

使用 `systemd-resolved` 管理 `dns`，配置 `/etc/systemd/resolved.conf`

```conf
DNS=vpn_dns_ip 8.8.8.8
```

重启服务

```bash
sudo systemctl restart systemd-resolved
```
