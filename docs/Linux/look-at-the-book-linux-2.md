# 看看书本上的linux系列-2

##  **进程的管理的权限与相关统计**

1. 运行统计信息, 包含用户/内核态运行时间; 上/下文切换次数; 启动时间等;
2. 进程亲缘关系
    * 拥有同一父进程的所有进程具有兄弟关系
    * 包含: 指向`parent`; 指向`real_parent`; 子进程双向链表头结点; 兄弟进程双向链表头结点
        * `parent`指向的父进程接收进程结束信号
        * `real_parent`和`parent`通常一样; 但在`bash`中用`GDB`调试程序时,`GDB`是`real_parent`,`bash`是`parent`
3. 进程权限, 包含`real_cred`指针(谁能操作我);`cred`指针(我能操作谁)
    * `cred`结构体中标明多组用户和用户组 id
    * `uid/gid`(哪个用户的进程启动我)
    * `euid/egid`(按照哪个用户审核权限, 操作消息队列, 共享内存等)
    * `fsuid/fsgid`(文件操作时审核)
    * 这三组`id`一般一样
    * 通过`chmod u+s program`, 给程序设置`set-user-id`标识位, 运行时程序将进程`euid/fsuid`改为程序文件所有者`id`
    * `suid/sgid`可以用来保存`id`, 进程可以通过`setuid`更改`uid`
    * `capability`机制, 以细粒度赋予普通用户部分高权限(`capability.h`列出了权限)
    * `cap_permitted`表示进程的权限
    * `cap_effective`实际起作用的权限,`cap_permitted`范围可大于`cap_effective`
    * `cap_inheritable`若权限可被继承, 在`exec`执行时继承的权限集合, 并加入`cap_permitted`中(但非`root`用户不会保`cap_inheritable`集合)
    * `cap_bset`所有进程保留的权限(限制只用一次的功能)
    * `cap_ambient_exec`时, 并入`cap_permitted`和`cap_effective`中
4. 内存管理:`mm_struct`
5. 文件与文件系统: 打开的文件, 文件系统相关数据结构
