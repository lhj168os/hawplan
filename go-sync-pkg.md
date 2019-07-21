# Golang sync包学习笔记

## ``type Once``
功能: 确保函数只执行一次

结构定义：
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

结构定义：
```go
type WaitGroup struct {
        // contains filtered or unexported fields
}

//function list:
// 计数器增加delta，表示增加delta个goroutine
func (wg *WaitGroup) Add(delta int)  

// 计数器减1，表示一个goroutine执行完成
func (wg *WaitGroup) Done()  

 // 挂起等待计数器归零        
func (wg *WaitGroup) Wait()         
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

结构定义：
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
## ``type RWMutex``
功能：
RWMutex 是单写多读锁，读锁占用的情况下会阻止写，不会阻止读，多个goroutine可以同时获取读锁，写锁会阻止其他goroutine（无论读和写）进来，整个锁由该 goroutine独占，适用于读多写少的场景.

PS：读共享，写独占，写优先

结构定义：
```go
type RWMutex struct {
        // contains filtered or unexported fields
}

//function list
func (rw *RWMutex) Lock()  // 写加锁
func (rw *RWMutex) RLock()  // 读加锁
// 返回一个实现Lock和Unlock方法的Locker接口，用于调用rw.RLock和rw.RUnlock来实现读锁的加锁和解锁。
func (rw *RWMutex) RLocker() Locker  
func (rw *RWMutex) RUnlock()  // 读解锁
func (rw *RWMutex) Unlock()  // 写解锁
```

示例：
```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

var value int

func writes(i int, wg *sync.WaitGroup, rwM *sync.RWMutex) {
	defer wg.Done()

	for {
		rand.Seed(time.Now().Unix())
		num := rand.Intn(100)
		rwM.Lock()
		value = num
		fmt.Printf("writes goroutine:%d, Write num=%d\n", i, num)
		rwM.Unlock()
		time.Sleep(time.Second * time.Duration(i+1))
	}
}

func readRLocker(i int, wg *sync.WaitGroup, rwM *sync.RWMutex) {
	defer wg.Done()
	w := rwM.RLocker()

	for {
		w.Lock()
		num := value
		fmt.Printf("readRLocker goroutine:%d, Reading num=%d\n", i, num)
		time.Sleep(time.Second * 2)
		w.Unlock()
		time.Sleep(time.Second * 1)
	}
}

func read2(i int, wg *sync.WaitGroup, rwM *sync.RWMutex) {
	defer wg.Done()

	for {
		rwM.RLock()
		num := value
		fmt.Printf("read2 goroutine:%d, Reading num=%d\n", i, num)
		time.Sleep(time.Second * 2)
		rwM.RUnlock()
		time.Sleep(time.Second * 1)
	}
}

func main() {
	rwMutex := &sync.RWMutex{}
	wg := &sync.WaitGroup{}

	for i := 0; i < 5; i++ {
		wg.Add(1)
		go writes(i, wg, rwMutex)
	}

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go readRLocker(i, wg, rwMutex)
	}

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go read2(i, wg, rwMutex)
	}

	wg.Wait()
}

```
输出：
```
writes goroutine:1, Write num=70
read2 goroutine:1, Reading num=70
read2 goroutine:6, Reading num=70
read2 goroutine:9, Reading num=70
read2 goroutine:7, Reading num=70
read2 goroutine:8, Reading num=70
read2 goroutine:4, Reading num=70
readRLocker goroutine:4, Reading num=70
readRLocker goroutine:1, Reading num=70
readRLocker goroutine:0, Reading num=70
readRLocker goroutine:8, Reading num=70
readRLocker goroutine:3, Reading num=70
readRLocker goroutine:2, Reading num=70
readRLocker goroutine:5, Reading num=70
readRLocker goroutine:9, Reading num=70
readRLocker goroutine:6, Reading num=70
read2 goroutine:2, Reading num=70
read2 goroutine:0, Reading num=70
read2 goroutine:3, Reading num=70
readRLocker goroutine:7, Reading num=70
read2 goroutine:5, Reading num=70
writes goroutine:0, Write num=70
writes goroutine:3, Write num=70
writes goroutine:2, Write num=70
writes goroutine:4, Write num=70
writes goroutine:1, Write num=93
readRLocker goroutine:5, Reading num=93
readRLocker goroutine:8, Reading num=93
read2 goroutine:4, Reading num=93
read2 goroutine:9, Reading num=93
read2 goroutine:7, Reading num=93
read2 goroutine:2, Reading num=93
read2 goroutine:8, Reading num=93
read2 goroutine:5, Reading num=93
readRLocker goroutine:0, Reading num=93
readRLocker goroutine:1, Reading num=93
readRLocker goroutine:3, Reading num=93
readRLocker goroutine:2, Reading num=93
readRLocker goroutine:4, Reading num=93
··· ···
```
---
## ``type Cond ``
功能：
>Broadcase、Signal
唤醒因wait condition而挂起goroutine，Signal只唤醒一个，而Broadcast唤醒所有。允许调用者获取基础锁Locker之后再调用唤醒，但非必需。

>Wait
必须获取该锁之后才能调用Wait()方法，Wait方法在调用时会释放底层锁Locker，并且将当前goroutine挂起，直到另一个goroutine执行Signal或者Broadcase，该goroutine才有机会重新唤醒，并尝试获取Locker，完成后续逻辑。

结构定义：
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
## ``type Pool``
功能：
Pool 是可伸缩、并发安全的临时对象池，用来存放已经分配但暂时不用的临时对象，通过对象重用机制，缓解 GC 压力，提高程序性能。

PS：如果你使用的Pool代码所需的东西不是大概同质的，那么从Pool中转化检索到所需要的内容的时间可能比重新实例化内容要花费的时间更多。

结构定义：
```go
type Pool struct {
        // New optionally specifies a function to generate
        // a value when Get would otherwise return nil.
        // It may not be changed concurrently with calls to Get.
        New func() interface{}
        // contains filtered or unexported fields
}

//function list:
func (p *Pool) Get() interface{}  //从 Pool 中获取元素，元素数量 -1，当 Pool 中没有元素时，会调用 New 生成元素，新元素不会放入 Pool 中，若 New 未定义，则返回 nil
func (p *Pool) Put(x interface{}) // 把用完的元素放回Pool中
```

示例：
```go
package poolTest

import (
	"fmt"
	"io/ioutil"
	"log"
	"net"
	"sync"
	"testing"
	"time"
)

func connectToService() interface{} {
	time.Sleep(1 * time.Second)
	return struct {}{}
}

func startNetworkDaemon() *sync.WaitGroup {
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		connPool := warmServiceConnCache()
		server, err := net.Listen("tcp", "localhost:8080")
		if err != nil {
			log.Fatalf("cannot listen: %v", err)
		}
		defer server.Close()
		wg.Done()

		for {
			conn, err := server.Accept()
			if err != nil {
				log.Printf("cannot accept connection: %v", err)
			}
			svcConn := connPool.Get()
			fmt.Fprintln(conn, "")
			connPool.Put(svcConn)
			conn.Close()
		}
	}()

	return &wg
}

func init() {
	daemonStarted := startNetworkDaemon()
	daemonStarted.Wait()
}

func BenchmarkNetworkRequest(b *testing.B) {
	for i := 0; i < b.N; i++ {
		conn, err := net.Dial("tcp", "localhost:8080")
		if err != nil {
			b.Fatalf("cannot dial host: %v", err)
		}
		if _, err := ioutil.ReadAll(conn); err != nil {
			b.Fatalf("cannot read: %v", err)
		}
		conn.Close()
	}
}

func warmServiceConnCache() *sync.Pool {
	p := &sync.Pool{
		New: connectToService,
	}

	for i := 0; i < 10; i++ {
		p.Put(p.New())
	}
	return p
}
```
输出：
```
goos: darwin
goarch: amd64
pkg: go-bfzd/poolTest
BenchmarkNetworkRequest-4           1000          10610437 ns/op
PASS
ok      go-bfzd/poolTest        38.099s

```
---
## ``type Map``
功能：
sync.Map这个数据结构是线程安全的（基本类型Map结构体在并发读写时会panic严重错误），它填补了Map线程不安全的缺陷，不过最好只在需要的情况下使用。它一般用于并发模型中对同一类map结构体的读写，或其他适用于sync.Map的情况。

结构定义：
```go
type Map struct {
        // contains filtered or unexported fields
}

//function list:
func (m *Map) Delete(key interface{})  // 删除对应key的值
func (m *Map) Load(key interface{}) (value interface{}, ok bool) // 获取key对应的值
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool)   //获取对应key的值，成功则返回true和value，不存在该key则将值存进该key并返回false和存进去的value
func (m *Map) Range(f func(key, value interface{}) bool)  // 遍历map中的所有key
func (m *Map) Store(key, value interface{})  // 将value存进对应的key，存在则覆盖
```

示例：
```go
// sync.Map是线程安全的，一般用于并发模型中，本例不存在并发，只演示各个接口的用法

package main

import (
	"fmt"
	"sync"
)

func main() {
	sMap := sync.Map{}

	fmt.Println("=====================Store=======================")

	fmt.Println("store: key=1, value=jay")
	sMap.Store(1, "jay")
	fmt.Println("store: key=2, value=lhj")
	sMap.Store(2, "lhj")
	fmt.Println("store: key=2, value=lhj-hhh")
	sMap.Store(2, "lhj-hhh")


	fmt.Println("=====================LoadOrStore=======================")

	result, succeed := sMap.LoadOrStore(1, "jays")
	fmt.Printf("key=1 store: value=jays; result: load_succeed=%t, value=%v\n", succeed, result)

	result, succeed = sMap.LoadOrStore(2, "lhj-s")
	fmt.Printf("key=2 store: value=lhj-s; result: load_succeed=%t, value=%v\n", succeed, result)

	result, succeed = sMap.LoadOrStore(3, "jay-lhj")
	fmt.Printf("key=3 store: value=jay-lhj; result: load_succeed=%t, value=%v\n", succeed, result)



	fmt.Println("=====================Load key:1~4=======================")

	if val, ok := sMap.Load(1); ok {
		fmt.Printf("key=1, value=%v\n", val)
	}else {
		fmt.Printf("key=1, load failed\n")
	}

	if val, ok := sMap.Load(2); ok {
		fmt.Printf("key=2, value=%v\n", val)
	}else {
		fmt.Printf("key=2, load failed\n")
	}

	if val, ok := sMap.Load(3); ok {
		fmt.Printf("key=3, value=%v\n", val)
	}else {
		fmt.Printf("key=3, load failed\n")
	}

	if val, ok := sMap.Load(4); ok {
		fmt.Printf("key=4, value=%v\n", val)
	}else {
		fmt.Printf("key=4, load failed\n")
	}

	fmt.Println("======================Delete======================")

	fmt.Println("delete key=1")
	sMap.Delete(1)

	fmt.Println("=====================Range=======================")
	sMap.Range(func(key, value interface{}) bool {
		fmt.Printf("key=%v value=%v\n", key, value)
		return true
	})

}
```

输出：
```
=====================Store=======================
store: key=1, value=jay
store: key=2, value=lhj
store: key=2, value=lhj-hhh
=====================LoadOrStore=======================
key=1 store: value=jays; result: load_succeed=true, value=jay
key=2 store: value=lhj-s; result: load_succeed=true, value=lhj-hhh
key=3 store: value=jay-lhj; result: load_succeed=false, value=jay-lhj
=====================Load key:1~4=======================
key=1, value=jay
key=2, value=lhj-hhh
key=3, value=jay-lhj
key=4, load failed
======================Delete======================
delete key=1
=====================Range=======================
key=2 value=lhj-hhh
key=3 value=jay-lhj
```
