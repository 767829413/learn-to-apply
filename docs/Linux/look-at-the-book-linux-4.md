# 看看书本上的linux系列-4

##  **调度, 切换运行进程, 有两种方式**

1. 进程调用`sleep`或等待`I/O`, 主动让出`CPU`
2. 进程运行一段时间, 被动让出`CPU`
    * 主动让出`CPU`的方式, 调用`schedule()`,`schedule()`调用`__schedule()`
    * `__schedule()`取出`rq`; 取出当前运行进程的`task_struct`
    * 调用`pick_next_task`取下一个进程
    * 依次调用调度类(优化: 大部分都是普通进程), 因此大多数情况调用`fair_sched_class`.`pick_next_task`
    * `pick_next_task_fair`先取出`cfs_rq`队列, 取出当前运行进程调度实体, 更新 vruntime
    * `pick_next_entity`取最左节点, 并得到`task_struct`, 若与当前进程不一样, 则更新红黑树`cfs_rq`
    * 进程上下文切换: 切换进程内存空间, 切换寄存器和`CPU`上下文(运行`context_switch`)
    * `context_switch()` -> `switch_to()` -> `__switch_to_asm`(切换(内核)栈顶指针) -> `__switch_to()`
    * `__switch_to()`取出`Per CPU`的`tss`(任务状态段)结构体,x86提供以硬件方式切换进程的模式, 为每个进程在内存中维护一个`tss`,`tss`有所有寄存器, 同时`TR`(任务寄存器)指向某个`tss`, 更改`TR`会触发换出`tss`(旧进程)和换入`tss`(新进程)
    * 切换进程没必要换所有寄存器,因此 Linux 中每个`CPU`关联一个`tss`, 同时`TR`不变,`Linux`中参与进程切换主要是栈顶寄存器
    * `task_struct` 的`thread`结构体保留切换时需要修改的寄存器, 切换时将新进程`thread`写入`CPU`的`tss`中
    * 具体各类指针保存位置和时刻
        * 用户栈, 切换进程内存空间时切换
        * 用户栈顶指针, 内核栈`pt_regs`中弹出
        * 用户指令指针, 从内核栈`pt_regs`中弹出
        * 内核栈, 由切换的`task_struct`中的`stack`指针指向
        * 内核栈顶指针,`__switch_to_asm`函数切换(保存在`thread`中)
        * 内核指令指针寄存器: 进程调度最终都会调用`__schedule`, 因此在让出(从当前进程)和取得(从其他进程)`CPU`时, 该指针都指向同一个代码位置
