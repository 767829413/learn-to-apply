# 如何在自定义对象上训练YOLOv5

## 对你自己的自定义对象进行训练

### 前提条件

* [Getting Started with Roboflow](https://blog.roboflow.com/getting-started-with-roboflow/)
* [LabelImg for Labeling Object Detection Data](https://blog.roboflow.com/labelimg/)
* [How to Train YOLO v5 on a Custom Dataset](https://www.youtube.com/watch?v=MdF6x6ZmLAY&ab_channel=Roboflow)
* [构建深度学习环境](../../docs/Play/python-cuda-opencv.md)

### 步骤一 克隆yolov5仓库,安装依赖

* 克隆仓库

    ```cmd
    # clone YOLOv5 repository
    git clone https://github.com/ultralytics/yolov5
    ```

* 安装依赖

    ```cmd
    cd yolov5
    pip install -qr requirements.txt
    ```

    通过 [构建深度学习环境](../../docs/Play/python-cuda-opencv.md) 后,可以不用执行依赖的安装

    环境是否ok验证

    ```python
    import torch
    from IPython.display import Image, clear_output
    print('Setup complete. Using torch %s %s' % (torch.__version__, torch.cuda.get_device_properties(0) if torch.cuda.is_available() else 'CPU'))
    ```

    显示这个表示ok

    ```text
    # %s 表示占位,实际输出为准
    Setup complete. Using torch %s.%s.%s+cux _CudaDeviceProperties(name='%s', major=%s, minor=%s, total_memory=%sMB, multi_processor_count=%s)
    ```

### 步骤二 下载格式正确的自定义数据集

* 下载自定义数据集

    从Roboflow下载数据集,导出格式使用 `YOLOv5 PyTorch`

    ![YOLOv5 PyTorch export](https://i.imgur.com/5vr9G2u.png)

    具体细节请看 `前提条件`

    将下载的文件解压后放到 `yolov5/mydata` (放哪随意)

    修改 `yolov5/mydata/data.yaml` 文件 `train` `val`

    ```yaml
    train: your_path/yolov5/mydata/train/images
    val: your_path/yolov5/mydata/valid/images

    nc: 1
    names: ['xxxx']
    ```

### 步骤三 定义模型配置和架构

* 修改 `yolov5/yolov5/models/` 文件夹下的模板

    这里使用的是 `yolov5s.yaml`,关于如何选择请[参考](https://github.com/ultralytics/yolov5/wiki/Train-Custom-Data)

    主要是修改 `nc` 为 `yolov5/mydata/data.yaml` 里定义的数量

    ```yaml
    # parameters
    nc: 80  # number of classes
    depth_multiple: 0.33  # model depth multiple
    width_multiple: 0.50  # layer channel multiple

    # anchors
    anchors:
        - [10,13, 16,30, 33,23]  # P3/8
        - [30,61, 62,45, 59,119]  # P4/16
        - [116,90, 156,198, 373,326]  # P5/32

    # YOLOv5 backbone
    backbone:
        # [from, number, module, args]
        [[-1, 1, Focus, [64, 3]],  # 0-P1/2
            [-1, 1, Conv, [128, 3, 2]],  # 1-P2/4
            [-1, 3, BottleneckCSP, [128]],
            [-1, 1, Conv, [256, 3, 2]],  # 3-P3/8
            [-1, 9, BottleneckCSP, [256]],
            [-1, 1, Conv, [512, 3, 2]],  # 5-P4/16
            [-1, 9, BottleneckCSP, [512]],
            [-1, 1, Conv, [1024, 3, 2]],  # 7-P5/32
            [-1, 1, SPP, [1024, [5, 9, 13]]],
            [-1, 3, BottleneckCSP, [1024, False]],  # 9
        ]

    # YOLOv5 head
    head:
        [[-1, 1, Conv, [512, 1, 1]],
            [-1, 1, nn.Upsample, [None, 2, 'nearest']],
            [[-1, 6], 1, Concat, [1]],  # cat backbone P4
            [-1, 3, BottleneckCSP, [512, False]],  # 13

            [-1, 1, Conv, [256, 1, 1]],
            [-1, 1, nn.Upsample, [None, 2, 'nearest']],
            [[-1, 4], 1, Concat, [1]],  # cat backbone P3
            [-1, 3, BottleneckCSP, [256, False]],  # 17 (P3/8-small)

            [-1, 1, Conv, [256, 3, 2]],
            [[-1, 14], 1, Concat, [1]],  # cat head P4
            [-1, 3, BottleneckCSP, [512, False]],  # 20 (P4/16-medium)

            [-1, 1, Conv, [512, 3, 2]],
            [[-1, 10], 1, Concat, [1]],  # cat head P5
            [-1, 3, BottleneckCSP, [1024, False]],  # 23 (P5/32-large)

            [[17, 20, 23], 1, Detect, [nc, anchors]],  # Detect(P3, P4, P5)
        ]
    ```

### 步骤四 开始训练 YOLOv5 Detector

* 使用 `yolov5/train.py` 来进行训练

    ```cmd
    python train.py --img 416 --batch 16 --epochs 800 --data your_path/yolov5/mydata/data.yaml --cfg /your_path/yolov5/models/custom_yolov5s.yaml --weights '' --workers 0 --name result  --cache
    ```

    训练成功会出现

    ```text
    ...skip
    
    Stopping training early as no improvement observed in last 800 epochs. Best results observed at epoch 482, best model saved as best.pt.
    
    ...skip

    Results saved to runs/train/mydata
    CPU times: user 12.3 s, sys: 1.59 s, total: 13.9 s
    Wall time: 14min 7s
    ```

### 步骤五 用训练过的模型进行推断

* 开始测试模型推断

    ```cmd
    python detect.py --weights  your_path/yolov5/runs/train/mydata/weights/best.pt --img 600 --conf 0.4 --source your_path/yolov5/test/images
    ```

    输出大概如下

    ```text
    ...skip

    Speed: 0.5ms pre-process, 13.7ms inference, 0.7ms NMS per image at shape (1, 3, 608, 608)
    Results saved to runs/detect/exp3
    ```

### 总结

**遇到问题建议谷歌,一般都是环境问题**
