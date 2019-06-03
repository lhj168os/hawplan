# Golang sync包学习笔记

## ``type Once``
功能: 确保函数只执行一次

函数原型：

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
## ``type Cond ``
功能：
```go
type Cond struct {

        // L is held while observing or changing the condition
        L Locker
        // contains filtered or unexported fields
}

//function list:
func NewCond(l Locker) *Cond
func (c *Cond) Broadcast()
func (c *Cond) Signal()
func (c *Cond) Wait()
```
---
## ``type Locker``
功能：
```go
type Locker interface {
        Lock()
        Unlock()
}
```
---
