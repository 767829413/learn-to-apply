
# 浅尝golang之Mutex基础

## **易错点**
* **Lock 和 Unlock 未成对使用**
* **复制已有的Mutex(goroutine使用Mutex是一个有状态的结构数据,复制这种有状态的结构本身就会引发预期之外的问题)**
* **Mutex不是可重入的(获取锁之后不能再次获取)**
* **死锁(互斥,持有与等待,不可释放,环装依赖)**

## **演示Code**
```
package base

import (
	"fmt"
	"sync"
	"time"
)

type UserError struct {
	sync.Mutex
	Num int
}

func UnUseLock() {
	var mu sync.Mutex
	defer mu.Unlock()
	fmt.Println("hello world!")
}

func CopyLock() {
	var u UserError
	u.Lock()
	defer u.Unlock()
	u.Num++
	copyLock(u)
}

func copyLock(u UserError) {
	u.Lock()
	defer u.Unlock()
	fmt.Println("in example")
}

func ReentrantLock(l sync.Locker) {
	defer l.Unlock()
	l.Lock()
	fmt.Println("out lock")
	reentrantLock(l)
}

func reentrantLock(l sync.Locker) {
	defer l.Unlock()
	l.Lock()
	fmt.Println("inner lock")
}

func DeadLock() {
	var lock1 sync.Mutex
	var lock2 sync.Mutex
	var wg sync.WaitGroup
	var resource int
	wg.Add(2)
	go func(resource int) {
		defer lock1.Unlock()
		defer wg.Done()
		lock1.Lock()
		fmt.Println("我要锁资源1")
		resource++
		time.Sleep(3 * time.Second)
		lock2.Lock()
		lock2.Unlock()

	}(resource)

	go func(resource int) {
		defer lock2.Unlock()
		defer wg.Done()
		lock2.Lock()
		fmt.Println("我要锁资源2")
		resource++
		time.Sleep(3 * time.Second)
		lock1.Lock()
		lock1.Unlock()
	}(resource)

	wg.Wait()

}
```

最后来个测试 code
```
package base

import (
	"sync"
	"testing"
)

const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexWaiterShift = iota
)

func TestGeneralCountTest(t *testing.T) {
	GeneralCount()
}

func TestPrintConst(t *testing.T) {
	t.Log(mutexLocked)
	t.Log(mutexWoken)
	t.Log(1 << mutexWaiterShift)
}

func TestGetGoId(t *testing.T) {
	t.Log(GetGoId())
}

func TestUnUseLock(t *testing.T) {
	UnUseLock()
}

func TestCopyLock(t *testing.T) {
	CopyLock()
}

func TestReentrantLock(t *testing.T) {
	l := &sync.Mutex{}
	ReentrantLock(l)
}
func TestDeadLock(t *testing.T) {
	DeadLock()
}
再来一些Mutex的可重入实现和使用样例

package base

import (
	"fmt"
	myGoid "github.com/petermattis/goid"
	"runtime"
	"strconv"
	"strings"
	"sync"
	"sync/atomic"
)

func GeneralCount() {
	var count Counter
	var wg sync.WaitGroup
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < 100000; j++ {
				count.Incr()
			}
		}()
	}
	wg.Wait()
	fmt.Println(count.Count())
}

type Counter struct {
	CounterType int
	Name        string

	mu    sync.Mutex
	count uint64
}

func (c *Counter) Incr() {
	defer c.mu.Unlock()
	c.mu.Lock()
	c.count++
}

func (c *Counter) Count() uint64 {
	defer c.mu.Unlock()
	c.mu.Lock()
	return c.count
}

func GetGoId() (int, error) {
	var buf [64]byte
	n := runtime.Stack(buf[:], false)
	str := strings.Split(string(buf[:n]), " ")
	return strconv.Atoi(str[1])
}

type RecursiveMutexGoId struct {
	sync.Mutex
	owner     int64 //当前 goroutine 的 id
	recursion int32 //可重入次数
}

func (r *RecursiveMutexGoId) Lock() {
	goId := myGoid.Get()
	//如果当前持有锁的goroutine就是这次调用的goroutine,说明是重入
	if atomic.LoadInt64(&r.owner) == goId {
		r.recursion++
		return
	}
	r.Mutex.Lock()
	//如果当前持有锁的goroutine就是这次调用的goroutine,说明是重入
	atomic.StoreInt64(&r.owner, goId)
	r.recursion = 1
}

func (r *RecursiveMutexGoId) Unlock() {
	goId := myGoid.Get()
	//非持有锁的goroutine尝试释放锁，错误的使用
	if atomic.LoadInt64(&r.owner) != goId {
		panic(fmt.Sprintf("wrong the owner(%d): %d!", r.owner, goId))
	}
	//调用次数减一
	r.recursion--
	//未完全释放,直接返回
	if r.recursion != 0 {
		return
	}
	atomic.StoreInt64(&r.owner, 0)
	r.Mutex.Unlock()
}

type RecursiveMutexToken struct {
	sync.Mutex
	token     int64 //自定义 goroutine 的标识
	recursion int32 //可重入次数
}

func (r *RecursiveMutexToken) Lock(token int64) {
	//如果传入的token和持有锁的token一致，说明是递归调用
	if atomic.LoadInt64(&r.token) == token {
		r.recursion++
		return
	}
	//传入的token不一致，说明不是递归调用
	r.Mutex.Lock()
	//抢到锁记录这个token
	atomic.StoreInt64(&r.token, token)
	r.recursion = 1
}

func (r *RecursiveMutexToken) Unlock(token int64) {
	if atomic.LoadInt64(&r.token) != token {
		panic(fmt.Sprintf("wrong the owner(%d): %d!", r.token, token))
	}
	r.recursion--
	if r.recursion != 0 {
		return
	}
	atomic.StoreInt64(&r.token, 0)
	r.Mutex.Unlock()
}
```

总结下就是使用Mutex要小心,稍不注意,就会翻车,特别是死锁哦(fatal error: all goroutines are asleep – deadlock!)