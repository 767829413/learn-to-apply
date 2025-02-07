# 配置 pip install 国内镜像源加速

## 说明
--------------------

因国内网络问题，直接 pip 安装包有时速度会非常慢，而且经常会出现装到一半失败了的问题。既然这样，我们就要充分利用国内镜像的力量，节省时间，明显提高 pip 安装的效率。

配置一般分为暂时置换和永久置换源镜像两种方法，选择哪一种视场景而定，不过一般推荐永久置换。

## 镜像源
--------------------

* 豆瓣 (douban) [http://pypi.douban.com/simple/](http://pypi.douban.com/simple/) (推荐)
* 阿里云 [http://mirrors.aliyun.com/pypi/simple/](http://mirrors.aliyun.com/pypi/simple/)
* 清华大学 [https://pypi.tuna.tsinghua.edu.cn/simple/](https://pypi.tuna.tsinghua.edu.cn/simple/)
* 中国科技大学 [https://pypi.mirrors.ustc.edu.cn/simple/](https://pypi.mirrors.ustc.edu.cn/simple/)
* 中国科学技术大学 [http://pypi.mirrors.ustc.edu.cn/simple/](http://pypi.mirrors.ustc.edu.cn/simple/)

推荐豆瓣 douban 镜像源，以下的所有示例也都会以豆瓣镜像源为例。

### 第一种：永久置换 pip 镜像源（推荐）

#### 1. 创建 pip.conf 文件

> 注意：如果你已经有 `~/.pip/pip.conf` 文件的话这步可以跳过。

运行以下命令：

```bash
cd ~/.pip
```

提示目录不存在的话就创建一个新的：

```bash
mkdir ~/.pip
cd ~/.pip
```

在 `.pip` 目录下创建一个 `pip.conf` 文件：

```bash
touch pip.conf
```

#### 2. 编辑 pip.conf 文件

使用 vim 或者任何编辑器修改此文件：

```bash
vim ~/.pip/pip.conf
```
    
新增如下内容：

```conf
    [global] 
    index-url=http://mirrors.aliyun.com/pypi/simple/ 
    [install] 
    trusted-host=mirrors.aliyun.com
```

#### 3. 测试一下

```bash
Collecting numpy (from -r requirements.txt (line 4))
  Downloading http://mirrors.aliyun.com/pypi/packages/9e/3e/3757f304c704f2f0294a6b8340fcf2be244038be07da4cccf390fa678a9f/numpy-2.1.3-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (16.0 MB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 16.0/16.0 MB 18.4 MB/s eta 0:00:00
Requirement already satisfied: charset-normalizer<4,>=2 in ./venv/lib/python3.12/site-packages (from requests->basicsr>=1.4.2->-r requirements.txt (line 1)) (3.4.1)
```

### 暂时置换 pip 镜像源

这里以豆瓣镜像源为例，接下来我们安装 `pygame` 包，也可以替换成你想安装的其他包名称：

```bash
pip install pygame -i http://pypi.douban.com/simple
```

这步如果出错，请将命令变换为：

```bash
pip install pygame -i http://pypi.douban.com/simple --trusted-host pypi.douban.com
```

安装效果：

```bash
pip install pygame -i http://pypi.douban.com/simple
Looking in indexes: http://pypi.douban.com/simple
Collecting pygame
    Downloading http://pypi.doubanio.com/packages/fe/830d313164c4f693892047327223775d3112fdb869900f2754bd134d7e76cc/pygame-1.94-cp27-cp27m-macosx_10_11_intel.whl (4.9MB)
    100% |████████████████████████████████| 4.9MB 10.3MB/s
Installing collected packages: pygame
Successfully installed pygame-1.9.4
```