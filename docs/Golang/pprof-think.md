# 调优方面的总结

## CPU使用过高

* 应用逻辑问题
  * json序列化
    * 使用一些优化过的json库替代
    * 使用二进制编码方式代替json编码
    * 同物理节点通信,使用共享内存IPC,直接干掉序列化开销
  * MD5 计算 hash 成本太高,可以使用 cityhash, murmurhash替代 
  * 其他的针对特定 pprof 来进行分析解决

* GC 使用 CPU 过高
  * 减少堆上对象分配
    * sync.Pool 进行堆对象的重用
    * 利用 slice 来替换 map 使用
    * 非指针(对象)替换指针(对象)
    * 小对象合并成一个大对象
  * offheap(黑科技,有点不太好把控)
  * 降低 GC 频率
    * 修改 GOGC (建议谷歌)
    * make 全局大slice (具体谷歌)

* 调度相关函数使用 CPU 过高
  * 尝试使用 goroutine pool 来减小 goroutine 的创建销毁
  * 控制最大 goroutine 数量

## 内存使用过高

* 堆内存使用过多
  * sync.Pool 对象复用 
  * 分级,为不同大小的对象提供不同等级的 sync.Pool
  * offheap(黑科技,有点不太好把控)

* goroutine 的栈占用过多内存
  * 减少 goroutine 数量
    * 例如连接本来是读 goroutine ,写 goroutine 可以变为 一个 goroutine 进行读写
    * 使用 goroutine pool 来限制最大 goroutine 数
    * 使用对应的 epoll 库(evio,gev 等)修改网络编程方式(适用于延迟不敏感业务) 
  * 修改代码,减少函数调用的层级(大概率是golang runtime 层级源码修改,很困难,代价高)

## 阻塞问题

* 上游系统问题
  * 找上游解决

* 锁的阻塞
  * 减少临界区范围
  * 降低锁的粒度
    * Global lock -> sharded lock
    * Global lock -> connection level lock
    * connection level lock -> request level lock
* 同步改异步
  * 日志场景: 同步日志 -> 异步日志
  * Metrics 上报场景: select -> select+default

* 个别场景使用双 buffer 来解决阻塞
