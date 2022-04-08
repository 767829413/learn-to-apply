
# 简单看下异步编程的 Promise 方法论(Golang实现)

## **promise.go**

```go
package promise

import "sync"

type Promise struct {
 wg  sync.WaitGroup
 res string
 err error
}

func NewPromise(f func() (string, error)) *Promise {
 p := &Promise{}
 p.wg.Add(1)
 go func() {
  p.res, p.err = f()
  p.wg.Done()
 }()
 return p
}

func (p *Promise) Then(r func(string), e func(error)) *Promise {
 go func() {
  p.wg.Wait()
  if p.err != nil {
   e(p.err)
   return
  }
  r(p.res)
 }()
 return p
}
```

## **easy_test.go**

简单的测试一个demo

```go
package promise

import (
 "fmt"
 "math/rand"
 "testing"
 "time"
)

func TestTicker(t *testing.T) {
 p := NewPromise(exampleTicker)
 p.Then(func(res string) {
  fmt.Println(res)
 }, func(err error) {
  fmt.Println(err)
 })
 p.wg.Wait()
}

func exampleTicker() (string, error) {
 for i := 0; i < 3; i++ {
  fmt.Println(i)
  <-time.Tick(time.Second * 1)
 }
 rand.Seed(time.Now().UTC().UnixNano())
 r := rand.Intn(100) % 2
 fmt.Println(r)
 if r != 0 {
  return "hello, world", nil
 } else {
  return "", fmt.Errorf("error")
 }
}
```
