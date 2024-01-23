# GitHub 加速终极教程

## 前置条件

* 木弟子要能用

## 设置 Http Proxy

```bash
$ git config --global http.proxy socks5h://127.0.0.1:7890
```

## 配置config

```bash
$ cat ~/.ssh/config

Host github.com
 Hostname ssh.github.com
 IdentityFile /xxx/.ssh/github_id_rsa
 User git
 Port 443
 ## linux 
 ## ProxyCommand nc -v -x 127.0.0.1:7890 %h %p
 ProxyCommand nc -v -x 127.0.0.1:7890 %h %p
```

## 扩展其他方案

* 通过https的方式访问仓库:
  * github的personal access token来进行验证
  * 通过git-credential-manager这个项目来对自己的token进行保存连接
  * 注意密钥交换的安全性问题