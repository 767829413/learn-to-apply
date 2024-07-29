# golang简易编译打包工具使用

## GOX

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

## GoReleaser

```bash
go install github.com/goreleaser/goreleaser@latest
```

生成基础配置

```bash
goreleaser init
```

```yaml
# This is an example .goreleaser.yml file with some sensible defaults.
# Make sure to check the documentation at https://goreleaser.com

# The lines below are called `modelines`. See `:help modeline`
# Feel free to remove those if you don't want/need to use them.
# yaml-language-server: $schema=https://goreleaser.com/static/schema.json
# vim: set ts=2 sw=2 tw=0 fo=cnqoj

version: 2

before:
  hooks:
    # You may remove this if you don't use go modules.
    - go mod tidy
    # you may remove this if you don't need go generate
    - go generate ./...

builds:
  - env:
      - CGO_ENABLED=0
    goos:
      - linux
      - windows
      - darwin

archives:
  - format: tar.gz
    # this name template makes the OS and Arch compatible with the results of `uname`.
    name_template: >-
      {{ .ProjectName }}_
      {{- title .Os }}_
      {{- if eq .Arch "amd64" }}x86_64
      {{- else if eq .Arch "386" }}i386
      {{- else }}{{ .Arch }}{{ end }}
      {{- if .Arm }}v{{ .Arm }}{{ end }}
    # use zip for windows archives
    format_overrides:
      - goos: windows
        format: zip

changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
```

1. version: 2: 指定使用 GoReleaser 配置文件的版本 2。
2. before.hooks: 在构建之前执行的命令。
    * go mod tidy: 整理 Go 模块依赖。
    * go generate ./...: 运行所有的 go generate 命令。
3. builds: 定义如何构建项目。
    * env: 设置环境变量，这里禁用了 CGO。
    * goos: 指定目标操作系统，包括 linux、windows 和 darwin（macOS）。
4. archives: 定义如何打包构建的二进制文件。
    * format: 默认使用 tar.gz 格式。
    * name_template: 定义生成的文件名模板。
    * format_overrides: 为 Windows 平台使用 zip 格式。
5. changelog: 配置变更日志生成。
    * sort: asc: 按升序排列变更。
    * filters.exclude: 排除某些类型的提交信息，如文档更新和测试。

临时构建用于测试 GoReleaser 的配置和构建

```bash
goreleaser release --snapshot --rm-dist
```

这是构建就加入ci配置

github

```bash
release:
  gitlab:
    owner: your-gitlab-username-or-group
    name: your-repo-name
```

gitlab

```bash
release:
  stage: release
  image: goreleaser/goreleaser
  script:
    - goreleaser release
  only:
    - tags
```

