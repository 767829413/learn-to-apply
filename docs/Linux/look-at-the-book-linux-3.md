# 看看书本上的linux系列-3

## **用户态/内核态切换执行如何串起来**

1. 用户态函数栈(通过`JMP`+ 参数 + 返回地址 调用函数)
2. 栈内存空间从高到低增长
3. 32位栈结构: 栈帧包含 前一个帧的`EBP`+ 局部变量 + N个参数 + 返回地址
    * `ESP`: 栈顶指针;`EBP`: 栈基址(栈帧最底部, 局部变量起始)
    * 返回值保存在`EAX`中
3. 64位栈结构: 结构类似
    * `rax`保存返回结果;`rsp`栈顶指针;`rbp`栈基指针
    * 参数传递时, 前 6个放寄存器中(再由被调用函数`push`进自己的栈, 用以寻址), 参数超过6个压入栈中
4. 内核栈结构
    * Linux为每个`task`分配了内核栈, 32位(8K), 64位(16K)
    * 栈结构: [预留8字节 +] `pt_regs` + 内核栈 + 头部`thread_info`
    * `thread_info`是`task_struct`的补充, 存储于体系结构有关的内容
    * `pt_regs`用以保存用户运行上下文, 通过`push`寄存器到栈中保存
5. 通过`task_struct`找到内核栈
    * 直接由`task_struct`内的`stack`直接得到指向`thread_info`的指针
    * 通过内核栈找到`task_struct`
    * 32位 直接由`thread_info`中的指针得到
    * 64位 每个`CPU`当前运行进程的`task_struct`的指针存放到`Per CPU`变量`current_task`中; 可调用`this_cpu_read_stable`进行读取
