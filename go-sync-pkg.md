# Golang sync包学习笔记

## ``type Once``
功能: 确保函数只执行一次

模块定义：
```go
type Once struct {
        // contains filtered or unexported fields
}

//function list：
func (o *Once) Do(f func())
```

用法示例：
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	var two sync.Once
	onceBody := func() {
		fmt.Println("Only once")
	}
	twoBody := func() {
		fmt.Println("two once")
	}
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			once.Do(onceBody)
			done <- true
		}()
	}
	for i := 0; i < 10; i++ {
		<-done
	}
	two.Do(twoBody)
	two.Do(twoBody)
}
```

输出：
```
Only once
two once
```
---
## ``type WaitGroup``
功能：
用于等待协程结束，每起一个goroutine之前Add(1),每个goroutine完成退出前Done()，主线程中用Wait()阻塞等待所有goroutine退出。

模块定义：
```go
type WaitGroup struct {
        // contains filtered or unexported fields
}

//function list:
func (wg *WaitGroup) Add(delta int)  // 计数器增加delta，表示增加delta个goroutine
func (wg *WaitGroup) Done()          // 计数器减1，表示一个goroutine执行完成
func (wg *WaitGroup) Wait()          // 挂起等待计数器归零
```
示例：
```go
package main

import (
	"fmt"
	"sync"
)

func tt(wg *sync.WaitGroup, i int) {
	defer wg.Done()
	fmt.Printf("goroutine: %d\n", i)
}

func main() {
	wg := &sync.WaitGroup{}
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go tt(wg, i)

	}
	wg.Wait()
	fmt.Println("Run end!")
}
```
输出：
```
goroutine: 9
goroutine: 6
goroutine: 7
goroutine: 8
goroutine: 3
goroutine: 2
goroutine: 4
goroutine: 1
goroutine: 5
goroutine: 0
Run end!
```
---
## ``type Locker``
功能：
确保资源只被一个goroutine占有

模块定义：
```go
type Locker interface {
        Lock()     // 上锁，在已锁的状态下此函数阻塞，直到锁被释放
        Unlock()   // 用于释放锁，在无锁的状态下执行会报错
}
```

示例：
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func tt(w *sync.WaitGroup, l *sync.Mutex, i int) {
	defer func() {
		fmt.Printf("End Goroutine: %d\n", i)
		l.Unlock()
		w.Done()
	}()
	l.Lock()
	fmt.Printf("lock: %d\n", i)
	time.Sleep(time.Second)
}

func main() {
	l := &sync.Mutex{}
	w := &sync.WaitGroup{}
	for i := 0; i < 10; i++ {
		w.Add(1)
		go tt(w, l, i)
	}

	w.Wait()
	fmt.Println("Run end!")
}
```

输出：
```
lock: 1
End Goroutine: 1
lock: 9
End Goroutine: 9
lock: 2
End Goroutine: 2
lock: 3
End Goroutine: 3
lock: 4
End Goroutine: 4
lock: 5
End Goroutine: 5
lock: 6
End Goroutine: 6
lock: 7
End Goroutine: 7
lock: 8
End Goroutine: 8
lock: 0
End Goroutine: 0
Run end!
```
---
## ``type Cond ``
功能：
>Broadcase、Signal
唤醒因wait condition而挂起goroutine，Signal只唤醒一个，而Broadcast唤醒所有。允许调用者获取基础锁Locker之后再调用唤醒，但非必需。

>Wait
必须获取该锁之后才能调用Wait()方法，Wait方法在调用时会释放底层锁Locker，并且将当前goroutine挂起，直到另一个goroutine执行Signal或者Broadcase，该goroutine才有机会重新唤醒，并尝试获取Locker，完成后续逻辑。

模块定义：
```go
type Cond struct {
        // L is held while observing or changing the condition
        L Locker
        // contains filtered or unexported fields
}

//function list:
func NewCond(l Locker) *Cond
func (c *Cond) Broadcast()   // 广播唤醒所有wait
func (c *Cond) Signal()      // 唤醒其中一个wait
func (c *Cond) Wait()        // 进入等待状态，释放锁
```

示例：
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func tt(cd *sync.Cond, wg *sync.WaitGroup, i int) {
	defer wg.Done()
	cd.L.Lock()
	fmt.Printf("lock: %d\n", i)
	cd.Wait()
	fmt.Printf("wait end: %d\n", i)
	cd.L.Unlock()
}

func main() {
	l := &sync.Mutex{}
	w := &sync.WaitGroup{}
	cd := sync.NewCond(l)

	for i := 0; i < 10; i++ {
		w.Add(1)
		go tt(cd, w, i)
	}
	time.Sleep(time.Second)
	fmt.Println("Send 5 Signal:")
	for i := 0; i < 5; i++ {
		cd.Signal()   //唤醒一个wait中的goroutine
	}
	time.Sleep(time.Second)
	fmt.Println("Send Broadcast:")
	cd.Broadcast()  //广播唤醒所有wait中的goroutine
	w.Wait()
	fmt.Println("Run end!")
}
```
输出：
```
lock: 1
lock: 2
lock: 7
lock: 8
lock: 9
lock: 0
lock: 3
lock: 6
lock: 5
lock: 4
Send 5 Signal:
wait end: 7
wait end: 9
wait end: 2
wait end: 8
wait end: 1
Send Broadcast:
wait end: 4
wait end: 0
wait end: 3
wait end: 6
wait end: 5
Run end!
```
---
