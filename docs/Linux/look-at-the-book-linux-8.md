# 看看书本上的linux系列-8

##  **创建进程相关**

1. `fork -> sys_call_table`转换为`sys_fork()->_do_fork`
2. 创建进程做两件事: 复制初始化`task_struct`; 唤醒新进程
3. 复制并初始化`task_struct`,`copy_process()`
    * `dup_task_struct`: 分配`task_struct`结构体; 创建内核栈, 赋给`* stack`; 复制`task_struct`, 设置`thread_info`
    * `copy_creds`: 分配`cred`结构体并复制,`p->cred = p->real_cred = get_cred(new)`
    * 初始化运行时统计量
    * `sched_fork`调度相关结构体: 分配并初始化`sched_entity`;`state = TASK_NEW`;设置优先级和调度类;`task_fork_fair()->update_curr`更新当前进程运行统计量, 将当前进程`vruntime`赋给子进程, 通过`sysctl_sched_child_runs_first`设置是否让子进程抢占, 若是则将其`sched_entity`放前头, 并调用`resched_curr`做被抢占标记.
    * 初始化文件和文件系统变量
    * `copy_files`: 复制进程打开的文件信息, 用`files_struct`维护
    * `copy_fs`: 复制进程目录信息, 包括根目录/根文件系统;`pwd`等, 用`fs_struct`维护
    * 初始化信号相关内容: 复制信号和处理函数
    * 复制内存空间: 分配并复制`mm_struct`; 复制内存映射信息
    * 分配`pid`
4. 唤醒新进程`wake_up_new_task()`
    * `state = TASK_RUNNING`;`activate`用调度类将当前子进程入队列
    * `enqueue_entiry`中会调用`update_curr`更新运行统计量, 再加入队列
    * 调用`check_preempt_curr`看是否能抢占, 若`task_fork_fair`中已设置`sysctl_sched_child_runs_first`, 直接返回, 否则进一步比较并调用`resched_curr`做抢占标记
    * 若父进程被标记会被抢占, 则系统调用`fork`返回过程会调度子进程
