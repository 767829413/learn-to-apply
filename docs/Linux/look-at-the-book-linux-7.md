# 看看书本上的linux系列-7

## **线程与进程的统辖**

1. 内核中进程, 线程统一为任务, 由`taks_struct`表示
2. 通过链表串起`task_struct`
3. `task_struct`中包含: 任务ID; 任务状态; 信号处理相关字段; 调度相关字段; 亲缘关系; 权限相关; 运行统计; 内存管理; 文件与文件系统; 内核栈
4. 任务 ID; 包含 `pid`, `tgid` 和 `group_leader`
    * `pid`(`process id`, 线程的id); `tgid`(`thread group id`, 所属进程(主线程)的id);`group_leader`指向`tgid`的结构体
    * 通过对比`pid`和`tgid`可判断是进程还是线程
    * 信号处理, 包含阻塞暂不处理; 等待处理; 正在处理的信号
    * 信号处理函数默认使用用户态的函数栈, 也可以开辟新的栈专门用于信号处理, 由`sas_ss_xxx`指定
    * 通过`pending/shared_pending`区分进程和线程的信号
5. 任务状态; 包含`state` `exit_state` `flags`
    * 准备运行状态`TASK_RUNNING`
    * 睡眠状态：可中断; 不可中断; 可杀
    * 可中断`TASK_INTERRUPTIBLE`, 收到信号要被唤醒
    * 不可中断`TASK_UNINTERRUPTIBLE`, 收到信号不会被唤醒, 不能被`kill`, 只能重启
    * 可杀`TASK_KILLABLE`, 可以响应致命信号, 由不可中断与`TASK_WAKEKILL`组合
    * 停止状态`TASK_STOPPED`, 由信号`SIGSTOP`,`SIGTTIN`,`SIGTSTP`与`SIGTTOU`触发进入
    * 调试跟踪`TASK_TRACED`， 被`debugger`等进程监视时进入
    * 结束状态(包含`exit_state`)
    * `EXIT_ZOMBIE`, 父进程还没有 `wait()`
    * `EXIT_DEAD`, 最终状态
    * `flags`, 例如`PF_VCPU`表示运行在虚拟`CPU`上;`PF_FORKNOEXEC` `_do_fork`函数里设置,`exec`函数中清除
    * 进程调度; 包含 是否在运行队列; 优先级; 调度策略; 可以使用那些`CPU`等信息.
