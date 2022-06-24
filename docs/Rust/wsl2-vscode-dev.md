# WSL2 + VSCode + Rust 开发环境搭建

## 在Win11快速搭建Rust开发环境

### 前提条件

* [已经安装好WSL2](../../docs/Productivity/wsl2-dev.md)
* [已经安装了VSCode](https://code.visualstudio.com/docs/setup/windows)

### 步骤一 安装 rust

在Windows Terminal打开Ubuntu 20.04执行一下命令,根据提示执行相应操作

```bash
## 出现 Rust is installed now. Great! 表示安装成功了
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
## 运行下面命令,出现 cargo 1.61.0 (a028ae4 2022-04-29) 表示安装成功
## zsh 建议在 .zshrc 文件中配置 . "$HOME/.cargo/env"
cargo version
## 如果是旧版需要更新
rustup update
```

### 步骤二 安装 gcc

Rust是必须要有C支持的,GCC必装

```bash
## 保证源最新
apt-get update
## 安装gcc
sudo apt install gcc
```

### 步骤三 打开vscode安装相应插件

主要安装

* [rust-analyzer](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust-analyzer)

* [Remote - WSL](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-wsl)

* [rust-analyzer(可选)](https://marketplace.visualstudio.com/items?itemName=dustypomerleau.rust-syntax)

### 步骤四 hello world

```bash
## 初始化一个 Rust 项目
cargo init hello-world
## 进入 hello-world 文件夹
cd ./hello-world
## 执行 
cargo run
#   Compiling hello-world v0.1.0 (/root/rust/hello-world)
#    Finished dev [unoptimized + debuginfo] target(s) in 0.59s
#     Running `target/debug/hello-world`
#Hello, world!
```
