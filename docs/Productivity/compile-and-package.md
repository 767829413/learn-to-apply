# golang简易编译打包工具使用

## GOX使用

交叉编译工具可以编译各种的环境

```bash
go get github.com/mitchellh/gox
gox -build-toolchain
```

直接运行gox。程序会一口气生成17个文件
横跨windows,linux,mac,freebsd,netbsd五大操作系统

```bash
gox -osarch "windows/amd64 linux/amd64" 或
gox -os "windows linux" -arch amd64
```
