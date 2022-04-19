# Docker镜像和容器的导入与导出及迁移

## docker镜像的导出和导入

### 显示当前docker中的镜像

```bash
docker images
```

镜像列表如下：

```bash
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
pointsift            latest              90b2ef439b40        2 weeks ago         12.6GB
ubuntu               18.04               735f80812f90        4 weeks ago         83.5MB
```

### 导出镜像

```bash
# docker save -o <保存路径> <镜像名称:标签>
# 如把A机 ubuntu:18.04 导出到当前文件夹，则在A机上运行
docker save -o ./ubuntu18.tar ubuntu:18.04
```

### 导入镜像

```bash
# 把A机当前文件夹下的ubuntu18.tar拷贝到另一台安装过docker的B机上
# 在B机上执行下述命令导入镜像,镜像ubuntu:18.04就成功的从A机复制到B机上了
docker load --input ./ubuntu18.tar
```

## docker容器的导出与导入

### 显示当前docker中运行的容器

```bash
docker ps
```

运行的容器列表如下

```bash
CONTAINER ID        IMAGE               COMMAND             CREATED         STATUS              PORTS               NAMES
4a02996e83b1        ubuntu:18.04        "/bin/bash"        44 secondsago    Up 42 seconds                           ubuntu18
```

### 停止容器

```bash
# 导出容器必须先停止容器
# docker stop <容器名>
# 如要想要导出ubuntu18,必须先停止(如果ubuntu18没有运行，则不需要执行此步骤)
docker stop ubuntu18
# 如果容器已经停止了，想要查看该容器，可以运行(该命令会显示所有的容器，包括运行的和非运行的)
docker ps -a
```

### 导出容器

```bash
# docker export <容器名> > <保存路径>
# 比如A机中导出容器ubuntu18
docker export ubuntu18 > ./ubuntu18.tar
```

### 导入容器

```bash
# docker import <文件路径> <容器名>
# A机当前文件夹下的ubuntu18.tar文件拷贝到B机上
# 在B机上运行下述命令
docker import ./ubuntu18.tar ubuntu18
```

### 启动容器

```bash
# docker start <容器名>
# 启动导入B机上的容器
docker start ubuntu18
```
