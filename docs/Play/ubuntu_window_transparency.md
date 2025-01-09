# ubuntu 22.04 窗口透明化指南

嘿嘿，为啥有这个不用我多说了，目的也是为了更好的工作哦

主要逻辑就是更改为所需的透明度级别，每次尝试 +/- 100

```bash
#!/bin/bash

# 文件路径，用于存储当前透明度值
OPACITY_FILE="$HOME/.current_opacity"

# 初始化透明度文件（如果不存在）
if [ ! -f "$OPACITY_FILE" ]; then
    echo "1000" > "$OPACITY_FILE"
fi

# 读取当前透明度值
current_opacity=$(cat "$OPACITY_FILE")

# 根据参数调整透明度
if [ "$1" = "increase" ]; then
    new_opacity=$((current_opacity + 100))
elif [ "$1" = "decrease" ]; then
    new_opacity=$((current_opacity - 100))
else
    echo "Usage: $0 [increase|decrease]"
    exit 1
fi

# 确保透明度在有效范围内（0-10000）
if [ $new_opacity -lt 0 ]; then
    new_opacity=0
elif [ $new_opacity -gt 10000 ]; then
    new_opacity=10000
fi

# 更新透明度文件
echo $new_opacity > "$OPACITY_FILE"

# 应用新的透明度
xprop -f _NET_WM_WINDOW_OPACITY 32c -set _NET_WM_WINDOW_OPACITY `printf 0x%x $((0xfffffff * $new_opacity / 100))`

echo "透明度已调整为 $new_opacity/100"
```

当然咯，我直接是设置好参数的:

```bash
#!/bin/bash

xprop -f _NET_WM_WINDOW_OPACITY 32c -set _NET_WM_WINDOW_OPACITY `printf 0x%x $((0xfffffff * 290 / 100))`
```

这个透明度很赞的

![01.png](https://s2.loli.net/2025/01/09/JSK7iINOEgFDnUA.png)

后面就绑定ubuntu的快捷键咯

![02.png](https://s2.loli.net/2025/01/09/iuVrOkTm13jSlxM.png)
