# 看看书本上的linux系列-1

##  **linux内核初始化**

linux内核初始化, 运行 `start_kernel()` 函数(位于 init/main.c)

---

### **创建样板进程, 及各个模块初始化**

1. 创建第一个进程, 0号进程. `set_task_stack_end_magic(&init_task)` and `struct task_struct init_task = INIT_TASK(init_task)`
2. 初始化中断, `trap_init()`. 系统调用也是通过发送中断进行, 由 `set_system_intr_gate()` 完成.
3. 初始化内存管理模块, `mm_init()`
4. 初始化进程调度模块, `sched_init()`
5. 初始化基于内存的文件系统 rootfs, `vfs_caches_init()`
    * [VFS](https://zh.wikipedia.org/wiki/%E8%99%9B%E6%93%AC%E6%AA%94%E6%A1%88%E7%B3%BB%E7%B5%B1)将各种文件系统抽象成统一接口
6. 调用 `rest_init()` 完成其他初始化工作

---

### **创建管理/创建用户态进程的进程, 1号进程**

1. `rest_init()` 通过 `kernel_thread(kernel_init,...)` 创建 1号进程(工作在用户态).
2. 权限管理
    * -x86 提供 4个`Ring`分层权限
    * 操作系统利用:`Ring0`-内核态(访问核心资源); `Ring3`-用户态(普通程序)
3. 用户态调用系统调用: 用户态-系统调用-保存寄存器-内核态执行系统调用-恢复寄存器-返回用户态
4. 新进程执行`kernel_init`函数, 先运行`ramdisk`的`init`程序(位于内存中)
    * 首先加载 ELF 文件
    * 设置用于保存用户态寄存器的结构体
    * 返回进入用户态
    * `init`加载存储设备的驱动
5. kernel_init 函数启动存储设备文件系统上的`init`

---

### **创建管理/创建内核态进程的进程, 2号进程**

1. `rest_init()` 通过 `kernel_thread`(`kthreadd`,...) 创建 2号进程(工作在内核态).
2. `kthreadd` 负责所有内核态线程的调度和管理
