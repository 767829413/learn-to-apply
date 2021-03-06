# 构建深度学习环境

## Win11如何为 Python 3.6 安装和构建 tensorflow-gpu 2.4.0, Cuda Toolkit 11.0 cuDNN 8.0, OpenCV 4.5.2 PyTorch v1.7.1

### 前提条件

* `NVIDIA` 系列显卡
* [PyTorch 和 Cuda Toolkit 版本兼容性](https://pytorch.org/get-started/previous-versions/#conda-2)
* [关于 PyTorch 离线安装](https://blog.csdn.net/shenjianhua005/article/details/115742154)
* [pip 离线安装包](https://blog.51cto.com/u_15054047/4182743)
* [PyTorch 离线安装包(要用gpu记得选带cu1x.x的)](https://download.pytorch.org/whl/torch_stable.html)
* [了解 tensorflow 相关兼容性](https://www.tensorflow.org/install/source_windows)
* [Google搜索](https://www.google.com/)
* [How To Install and Build OpenCV with GPU for Python | VS Code | NVIDIA Cuda and OpenCV 4.5.2](https://www.youtube.com/watch?v=HsuKxjQhFU0&ab_channel=NicolaiNielsen-ComputerVision&AI)
* **信我,按照我这个版本来,你会很通畅,别的版本组合会让你有种便秘的感觉**

### 步骤一 安装 Visual Studio 2019

[Visual Studio 2019 下载地址](https://visualstudio.microsoft.com/zh-hans/vs/older-downloads/)

下载完成后点击安装

![20220710223852.png](https://s2.loli.net/2022/07/10/lNG8SjXZPQKC7Jk.png)

选择c++开发套件进行安装

![20220710223707.png](https://s2.loli.net/2022/07/10/XD2IJ7EtR3jY4Ld.png)

### 步骤二 安装 Anaconda3

***强烈建议python版本3.6的，相信我，用高版本安装的话，后续流程会吐的***

[Anaconda 3安装（Python 3.6版本）](https://repo.anaconda.com/archive/Anaconda3-5.2.0-Windows-x86_64.exe)

安装按照下一步进行就行了，建议最后将放入path选项勾上

```cmd
## 验证安装
python -V
Python 3.6.5 :: Anaconda, Inc.
```

国内网不好的,那就按照下面的操作

[pip conda 安装速度慢解决方法](https://cloud.tencent.com/developer/article/1695021)
[Anaconda 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/)

### 步骤三 安装 Cuda Toolkit 11.0, cuDNN 8.0

[Cuda Toolkit 版本地址](https://developer.nvidia.com/cuda-toolkit-archive)

[Cuda 11.0 下载地址](https://developer.nvidia.com/cuda-11-0-3-download-archive?target_os=Windows&target_arch=x86_64&target_version=10&target_type=exe_local)

[Cudnn 版本地址](https://developer.nvidia.com/rdp/cudnn-archive)

[Cudnn 8.0 下载地址](https://developer.nvidia.com/compute/machine-learning/cudnn/secure/8.0.5/11.0_20201106/cudnn-11.0-windows-x64-v8.0.5.39.zip)

下载完成后先安装 `Cuda 11.0`

安装完成后解压 `Cudnn 8.0`

![713231420.png](https://s2.loli.net/2022/07/13/uPqCSpcEJXVg14d.png)

将解压后的 `cuda文件` 移动到你指定的目录下 `your_path/cuda`

![343567711000831.png](https://s2.loli.net/2022/07/11/lXPFng3UE5whrCQ.png)

设置环境变量,更新到path下面, `your_path/cuda/bin`, `your_path/cuda/include`, `your_path/cuda/lib/x64`

![rfv-0711001029.png](https://s2.loli.net/2022/07/11/TflbjRkq9zLYadF.png)

### 步骤四 安装 tensorflow-gpu 2.4.0

首先配置环境变量

```cmd
conda create -n myenv python=3.6
Solving environment: done


==> WARNING: A newer version of conda exists. <==
  current version: 4.5.4
  latest version: 4.13.0

Please update conda by running

    $ conda update -n base conda



## Package Plan ##

  environment location: C:\Users\76782\Anaconda3\envs\myenv

  added / updated specs:
    - python=3.6


The following packages will be downloaded:

...

Preparing transaction: done
Verifying transaction: done
Executing transaction: done
#
# To activate this environment, use:
# > activate myenv
#
# To deactivate an active environment, use:
# > deactivate
#
# * for power-users using bash, you must source
#
```

环境变量配置结束后激活下

```cmd
conda activate myenv
```

正式开始安装 `tensorflow-gpu 2.4.0`

```cmd
pip install tensorflow-gpu==2.4.0
```

验证是否安装成功

```cmd
python
Python 3.6.5 |Anaconda, Inc.| (default, Mar 29 2018, 13:32:41) [MSC v.1900 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> tf.config.list_physical_devices('GPU')
[PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU')]
>>> tf.test.is_built_with_cuda()
True
>>>
```

***注意：***

1. 如果出现 `CondaHTTPError: HTTP 000 CONNECTION 问题` 建议看看这个 [解决方案](https://zhuanlan.zhihu.com/p/260034241)

修改 `Win11` 下的 C:\Users%user_name\.condarc

```cmd
channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/Paddle/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/fastai/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/bioconda/
show_channel_urls: true
ssl_verify: false
```

2. 如果出现 `pip版本过低` ,按照提示做就可以了

```text
protobuf requires Python '>=3.7' but the running Python is 3.6.5
You are using pip version 10.0.1, however version 21.3.1 is available.
You should consider upgrading via the 'python -m pip install --upgrade pip' command.
```

```cmd
python -m pip install --upgrade pip
```

3. 如果出现 `ERROR: Cannot uninstall 'wrapt'.`

```text
It is a distutils installed project and thus we cannot accurately determine which files belong to it which would lead to only a partial uninstall.`
```

[解决方案](https://www.cnblogs.com/conver/p/11141176.html)

```cmd
pip install -U --ignore-installed wrapt
```

### 步骤五 编译构建 OpenCV

首先下载源码

[OpenCV Source Code](https://github.com/opencv/opencv/tree/4.5.2)

[OpenCV Contrib](https://github.com/opencv/opencv_contrib/tree/4.5.2)

然后下载 `CMake`

[CMake](https://cmake.org/download/)

开始编译前准备工作

1. 构建编译目录

将下载好的 `OpenCV Source Code`, `OpenCV Contrib` 源码目录放到你准备构建编译的目录里,然后新建一个 `build` 文件夹

![fff-11002037.png](https://s2.loli.net/2022/07/11/baAxQmDeKiqyFrH.png)

2. 用 `CMake` 进行构建配置与二进制文件生成

先设置文件

![fff9-11002037.png](https://s2.loli.net/2022/07/11/JMQxfIz31el62HZ.png)

点击 `Configure` 进行配置

![fff-11002037.png](https://s2.loli.net/2022/07/11/7q1pUZvBx3Kf42H.png)

查看输出内容

```text
...skip

  OpenCV modules:
    To be built:                 calib3d core dnn features2d flann gapi highgui imgcodecs imgproc ml objdetect photo stitching ts video videoio
    Disabled:                    world
    Disabled by dependency:      -
    Unavailable:                 java python2 python3
    Applications:                tests perf_tests apps
    Documentation:               NO
    Non-free algorithms:         NO

...skip

  Python (for build):            NO

  Java:                          
    ant:                         NO
    JNI:                         NO
    Java wrappers:               NO
    Java tests:                  NO

  Install to:                    C:/Users/76782/Documents/opencv_python/build/install
-----------------------------------------------------------------

Configuring done
```

`Unavailable:java python2 python3` 和 `Python (for build):NO` 表示未识别支持 `Python` 环境

可以搜索 `python` 可以自行配置

| NAME | value |
| :---- | :----- |
| PYTHON3_EXECUTABLE | `your_path/Anaconda3/python.exec` |
| PYTHON3_INCLUDE_DIR | `your_path/Anaconda3/include` |
| PYTHON3_LIBRARY | `your_path/Anaconda3/libs/python36.lib` |
| PYTHON3_LIBRARY_DEBUG | PYTHON_DEBUG_LIBRARY-NOTFOUND |
| PYTHON3_NUMPY_INCLUDE_DIRS | `your_path/Anaconda3/Lib/site-packages/numpy/core/include` |
| PYTHON3_PACKAGES_PATH | `your_path/Anaconda3/Lib/site-packages` |

![qwe-11002037.png](https://s2.loli.net/2022/07/11/srTXB4v6jqNhJKQ.png)

如果还是不行,请看这个

[方案前置](https://stackoverflow.com/questions/24174394/cmake-is-not-able-to-find-python-libraries/33759845#33759845)

[方案](https://answers.opencv.org/question/229281/opencv-python-built-for-windows-x64-python-37-pyd-file-not-found-opencv-modules-unavailable/)

***以上都不行建议删掉build文件夹改下名字试试***

再次 `Configure` 查看输出 `Unavailable:java python2`表示成功

```text
...skip

  OpenCV modules:
    To be built:                 calib3d core dnn features2d flann gapi highgui imgcodecs imgproc ml objdetect photo python3 stitching ts video videoio
    Disabled:                    world
    Disabled by dependency:      -
    Unavailable:                 java python2
    Applications:                tests perf_tests apps
    Documentation:               NO
    Non-free algorithms:         NO

...skip
```

下面再进行一些配置

1. 搜索 `with_cuda`, 勾选 `WITH_CUDA`

![fff-11002037.png](https://s2.loli.net/2022/07/11/QVGZm3DfC1nLhPB.png)

2. 搜索 `opencv_dnn`, 勾选 `BUILD_opencv_dnn`, `OPENCV_DNN_CUDA`, `OPENCV_DNN_OPENCL`

![fff-11002037.png](https://s2.loli.net/2022/07/11/mVwkRlZyq7JCYpc.png)

3. 搜索 `fast`, 勾选 `ENABLE_FAST_MATH`

![fff-11002037.png](https://s2.loli.net/2022/07/11/XIFMuNROk28ZrpG.png)

4. 搜索 `world`, 勾选 `BUILD_opencv_world`

![fff-11002037.png](https://s2.loli.net/2022/07/11/OBptswH2lgC9e3z.png)

5. 搜索 `extra`, 设置 `your_path/opencv_contrib-4.5.2/modules`

![fff-11002037.png](https://s2.loli.net/2022/07/11/CD6T71iLh8R9PHg.png)

6. 再次 `Configure` 后,搜索 `fast`, 勾选 `CUDA_FAST_MATH`

![fff-11002037.png](https://s2.loli.net/2022/07/11/sXToABHeg7EKxMm.png)

7. 搜索 `arc`, 这里配置 `CUDA_ARCH_BIN`, 配置请参考 [Cuda Wikipedia](https://en.wikipedia.org/wiki/CUDA)

![fff-11002037.png](https://s2.loli.net/2022/07/11/AD4sxIU5TSZ8Ko2.png)

8. 搜索 `conf`,设置 `CMAKE_CONFIGURATION_TYPES` 为 `Release` (windows不要用 `Debug`)

![fff-11002037.png](https://s2.loli.net/2022/07/11/h8UM1jbZtxYadzG.png)

最后点击 `Generate` ,查看输出以下即可

```text
...skip
-----------------------------------------------------------------

Configuring done
Generating done
```

准备工作做完后就可以执行编译了,大概一两个小时

```cmd
## "C:\Users\76782\Documents\opencv_python\build" 换成你的路径
cmake --build "C:\Users\76782\Documents\opencv_python\build" --target INSTALL --config Release
```

完成后需要验证下是否成功

```cmd
python
Python 3.6.5 |Anaconda, Inc.| (default, Mar 29 2018, 13:32:41) [MSC v.1900 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
>>> from cv2 import cuda
>>> cuda.printCudaDeviceInfo(0)
*** CUDA Device Query (Runtime API) version (CUDART static linking) ***

Device count: 1

Device 0: "NVIDIA GeForce RTX 3060"
  CUDA Driver Version / Runtime Version          11.60 / 11.20
  CUDA Capability Major/Minor version number:    8.6
  Total amount of global memory:                 12288 MBytes (12884377600 bytes)
  GPU Clock Speed:                               1.78 GHz
  Max Texture Dimension Size (x,y,z)             1D=(131072), 2D=(131072,65536), 3D=(16384,16384,16384)
  Max Layered Texture Size (dim) x layers        1D=(32768) x 2048, 2D=(32768,32768) x 2048
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 65536
  Warp size:                                     32
  Maximum number of threads per block:           1024
  Maximum sizes of each dimension of a block:    1024 x 1024 x 64
  Maximum sizes of each dimension of a grid:     2147483647 x 65535 x 65535
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and execution:                 Yes with 1 copy engine(s)
  Run time limit on kernels:                     Yes
  Integrated GPU sharing Host Memory:            No
  Support host page-locked memory mapping:       Yes
  Concurrent kernel execution:                   Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support enabled:                No
  Device is using TCC driver mode:               No
  Device supports Unified Addressing (UVA):      Yes
  Device PCI Bus ID / PCI location ID:           1 / 0
  Compute Mode:
      Default (multiple host threads can use ::cudaSetDevice() with device simultaneously)

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version  = 11.60, CUDA Runtime Version = 11.20, NumDevs = 1

>>>
```

***注意：***

如果出现以下输出 `Download failed: 6;"Couldn't resolve host name"`,建议打开 `C:\Users\76782\Documents\opencv_python\build` 查看日志文件 `CMakeDownloadLog.txt` 有以下输出

```text
...skip
#try 1
# timeout on name lookup is not supported
# getaddrinfo(3) failed for raw.githubusercontent.com:443
# Could not resolve host: raw.githubusercontent.com
# Closing connection 0
# 
...skip
```

[解决方案](https://github.com/hawtim/hawtim.github.io/issues/10)

主要就是自行添加 `raw.githubusercontent.com` 在本地 `hosts` 的ip

### 总结

**建议不要用高版本 `Python`**

**建议一定要有 `梯子`**
