
# 浅尝golang之Cond和Once

## **演示Code**
```
type Queue struct {
   capacity int
   Data     []int
   c        *sync.Cond
}

func NewQueue(num int) *Queue {
   return &Queue{
      capacity: num,
      Data:     make([]int, 0, num),
      c:        sync.NewCond(&sync.Mutex{}),
   }
}

func (q *Queue) push(val int) {
   defer q.c.L.Unlock()
   q.c.L.Lock()
   for len(q.Data) == q.capacity {
      q.c.Wait()
   }
   q.Data = append(q.Data, val)
   q.c.Broadcast()
}

func (q *Queue) pop() int {
   defer q.c.L.Unlock()
   q.c.L.Lock()
   for len(q.Data) == 0 {
      q.c.Wait()
   }
   val := q.Data[0]
   q.Data = q.Data[1:]
   q.c.Broadcast()
   return val
}

// Athlete race
func ExampleOne() {
   c := sync.NewCond(&sync.Mutex{})
   defer c.L.Unlock()
   var ready int

   for i := 0; i < 10; i++ {
      go func(i int) {
         defer c.L.Unlock()
         time.Sleep(time.Duration(rand.Int63n(5)) * time.Second)
         c.L.Lock()
         ready++
         log.Printf("Athlete: %d Ready", i)
         //Wake up waiter
         c.Signal()
      }(i)
   }
   c.L.Lock()
   for ready != 10 {
      c.Wait()
      log.Printf("The referee is awakened")
   }
   log.Printf("Game start")
}
```
添加个测试用的code
```
func TestExampleOne(t *testing.T) {
   ExampleOne()
}

func TestMyCondQueue(t *testing.T) {
   q := NewQueue(5)
   go func() {
      for {
         q.push(rand.Intn(20))
      }

   }()
   for {
      t.Log(q.pop())
      time.Sleep(1 * time.Second)
   }
}
```
Once比较简单,单例模式惯用,配置初始化

```
type MyOnce struct {
   m    sync.Mutex
   done uint32
}

func (o *MyOnce) Do(f func() error) error {
   if atomic.LoadUint32(&o.done) == 1 {
      return nil
   }
   return o.doSlow(f)
}

func (o *MyOnce) doSlow(f func() error) error {
   defer o.m.Unlock()
   o.m.Lock()
   var err error
   if o.done == 0 {
      err = f()
      if err != nil {
         atomic.StoreUint32(&o.done, 1)
      }
   }
   return err
}
```

扩充下功能,方便判断是否成功