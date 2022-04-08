
# 浅尝golang之Mutex的扩展操作(重入,try lock,统计goroutine等待数)

## **演示Code**

```go
package base

import (
 "fmt"
 "sync"
 "sync/atomic"
 "unsafe"
)

const (
 mutexLocked = 1 << iota // mutex is locked
 mutexWoken
 mutexStarving
 mutexWaiterShift      = iota
 starvationThresholdNs = 1e6
)

type MyMutex struct {
 sync.Mutex
 token     int64 //自定义 goroutine 的标识
 recursion int32 //可重入次数
}

func (m *MyMutex) Lock(token int64) {
 //如果传入的token和持有锁的token一致，说明是递归调用
 if atomic.LoadInt64(&m.token) == token {
  m.recursion++
  return
 }
 //传入的token不一致，说明不是递归调用
 m.Mutex.Lock()
 //抢到锁记录这个token
 atomic.StoreInt64(&m.token, token)
 m.recursion = 1
}

func (m *MyMutex) Unlock(token int64) {
 if atomic.LoadInt64(&m.token) != token {
  panic(fmt.Sprintf("wrong the owner(%d): %d!", m.token, token))
 }
 m.recursion--
 if m.recursion != 0 {
  return
 }
 atomic.StoreInt64(&m.token, 0)
 m.Mutex.Unlock()
}

func (m *MyMutex) TryLock(token int64) bool {
 if atomic.LoadInt64(&m.token) == token {
  m.recursion++
  return true
 }
 //成功抢到锁
 if atomic.CompareAndSwapInt32((*int32)(unsafe.Pointer(&m.Mutex)), 0, mutexLocked) {
  //抢到锁记录这个token
  atomic.StoreInt64(&m.token, token)
  m.recursion = 1
  return true
 }
 // 处于唤醒,加锁,饥饿则不参加此次竞争
 oldV := atomic.LoadInt32((*int32)(unsafe.Pointer(&m.Mutex)))
 if oldV&(mutexLocked|mutexStarving|mutexWoken) != 0 {
  return false
 }
 //尝试竞争拿锁
 newV := oldV | mutexLocked
 if atomic.CompareAndSwapInt32((*int32)(unsafe.Pointer(&m.Mutex)), oldV, newV) {
  //抢到锁记录这个token
  atomic.StoreInt64(&m.token, token)
  m.recursion = 1
  return true
 }
 return false
}

func (m *MyMutex) WaitCount() int {
 //通过state字段来获取等待锁的 goroutine
 num := atomic.LoadInt32((*int32)(unsafe.Pointer(&m.Mutex)))
 num = num >> mutexWaiterShift
 num = num + (num & mutexLocked)
 return int(num)
}
```

重不重入我用的一个标识传递来实现标识当前goroutine,没有使用下面这种方式

```go
    n := runtime.Stack(buf[:], false)
  str := strings.Split(string(buf[:n]), false)
```

这种解析的不可控,还是传递标识比较稳定

最后来个测试 code

```go
package base

import (
 "fmt"
 "sync"
 "testing"
 "time"
)

func TestMyMutexTryLock(t *testing.T) {
 var myMutex MyMutex
 go func() {
  defer myMutex.Unlock(11)
  myMutex.Lock(11)
  fmt.Println("我锁了")
  time.Sleep(5 * time.Second)
 }()
 for {
  time.Sleep(1 * time.Second)
  if myMutex.TryLock(12) {
   fmt.Println("我拿锁了")
   myMutex.Unlock(12)
   break
  }
 }
 fmt.Println("end")
}

func TestMyMutexWaitCount(t *testing.T) {
 var wg sync.WaitGroup
 var myMutex MyMutex
 for i := 11; i < 17; i++ {
  wg.Add(1)
  go func(i int) {
   defer myMutex.Unlock(int64(i))
   defer wg.Done()
   myMutex.Lock(int64(i))
   if i == 11 {
    time.Sleep(2 * time.Second)
   }
  }(i)
 }
 time.Sleep(1 * time.Second)
 fmt.Println(6, myMutex.WaitCount())
 wg.Wait()
}
```
