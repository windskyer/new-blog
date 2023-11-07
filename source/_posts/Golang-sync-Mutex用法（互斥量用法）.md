---
title: Golang sync.Mutex用法（互斥量用法）
categories: Kubernetes
sage: false
date: 2020-04-18 18:01:58
tags: go
---

## 介绍
golang 中的 sync 包实现了两种锁：

>1. Mutex：互斥锁
>2. RWMutex：读写锁，RWMutex 基于 Mutex 实现

<!-- more -->

## Mutex（互斥锁）

>1. Mutex 为互斥锁，Lock() 加锁，Unlock() 解锁
>2. 在一个 goroutine 获得 Mutex 后，其他 goroutine 只能等到这个 goroutine 释放该 Mutex
>3. 使用 Lock() 加锁后，不能再继续对其加锁，直到利用 Unlock() 解锁后才能再加锁
>4. 在 Lock() 之前使用 Unlock() 会导致 panic 异常
>5. 已经锁定的 Mutex 并不与特定的 goroutine 相关联，这样可以利用一个 goroutine 对其加锁，再利用其他 goroutine 对其解锁
>6. 在同一个 goroutine 中的 Mutex 解锁之前再次进行加锁，会导致死锁
>7. 适用于读写不确定，并且只有一个读或者写的场景

## 示例

### 加锁和解锁示例
```go
package main

import (
    "time"
    "fmt"
    "sync"
)

func main() {
    var mutex sync.Mutex
    fmt.Println("Lock the lock")
    mutex.Lock()
    fmt.Println("The lock is locked")
    channels := make([]chan int, 4)
    for i := 0; i < 4; i++ {
        channels[i] = make(chan int)
        go func(i int, c chan int) {
            fmt.Println("Not lock: ", i)
            mutex.Lock()
            fmt.Println("Locked: ", i)
            time.Sleep(time.Second)
            fmt.Println("Unlock the lock: ", i)
            mutex.Unlock()
            c <- i
        }(i, channels[i])
    }
    time.Sleep(time.Second)
    fmt.Println("Unlock the lock")
    mutex.Unlock()
    time.Sleep(time.Second)

    for _, c := range channels {
        <-c
    }
}
```

程序输出：

```bash
Lock the lock
The lock is locked
Not lock:  1
Not lock:  2
Not lock:  0
Not lock:  3
Unlock the lock
Locked:  1
Unlock the lock:  1
Locked:  2
Unlock the lock:  2
Locked:  3
Unlock the lock:  3
Locked:  0
Unlock the lock:  0
```

### 在解锁之前加锁会导致死锁

```go
package main

import (
    "fmt"
    "sync"
)

func main(){
    var mutex sync.Mutex
    mutex.Lock()
    fmt.Println("Locked")
    mutex.Lock()
}
```

程序输出：
```bash
Locked
fatal error: all goroutines are asleep - deadlock!
```

## RWMutex（读写锁）

>1. RWMutex 是单写多读锁，该锁可以加多个读锁或者一个写锁
>2. 读锁占用的情况下会阻止写，不会阻止读，多个 goroutine 可以同时获取读锁
>3. 写锁会阻止其他 goroutine（无论读和写）进来，整个锁由该 goroutine 独占
>4. 适用于读多写少的场景

## Lock() 和 Unlock()

>1. Lock() 加写锁，Unlock() 解写锁
>2. 如果在加写锁之前已经有其他的读锁和写锁，则 Lock() 会阻塞直到该锁可用，为确保锁可用，已经阻塞的 Lock() 调用会从获得的锁中排除新的读取器，即写锁权限高于读锁，有写锁时优先进行写锁定
>3. 在 Lock() 之前使用 Unlock() 会导致 panic 异常

## RLock() 和 RUnlock()

>1. RLock() 加读锁，RUnlock() 解读锁
>2. RLock() 加读锁时，如果存在写锁，则无法加读锁；当只有读锁或者没有锁时，可以加读锁，读锁可以加载多个
>3. RUnlock() 解读锁，RUnlock() 撤销单词 RLock() 调用，对于其他同时存在的读锁则没有效果
>4. 在没有读锁的情况下调用 RUnlock() 会导致 panic 错误
>5. RUnlock() 的个数不得多余 RLock()，否则会导致 panic 错误

## 示例

### Lock() 和 Unlock()
```go
package main

import (
    "sync"
    "fmt"
    "time"
)

func main() {
    var mutex *sync.RWMutex
    mutex = new(sync.RWMutex)
    fmt.Println("Lock the lock")
    mutex.Lock()
    fmt.Println("The lock is locked")

    channels := make([]chan int, 4)
    for i := 0; i < 4; i++ {
        channels[i] = make(chan int)
        go func(i int, c chan int) {
            fmt.Println("Not lock: ", i)
            mutex.Lock()
            fmt.Println("Locked: ", i)
            fmt.Println("Unlock the lock: ", i)
            mutex.Unlock()
            c <- i
        }(i, channels[i])
    }
    time.Sleep(time.Second)
    fmt.Println("Unlock the lock")
    mutex.Unlock()
    time.Sleep(time.Second)

    for _, c := range channels {
        <-c
    }
}
```

程序输出：
```bash
Lock the lock
The lock is locked
Not lock:  0
Not lock:  1
Not lock:  2
Not lock:  3
Unlock the lock
Locked:  0
Unlock the lock:  0
Locked:  2
Unlock the lock:  2
Locked:  3
Unlock the lock:  3
Locked:  1
Unlock the lock:  1
```

### Lock() 和 RLock()

```go
package main

import (
    "sync"
    "fmt"
    "time"
)

func main() {
    var mutex *sync.RWMutex
    mutex = new(sync.RWMutex)
    fmt.Println("Lock the lock")
    mutex.Lock()
    fmt.Println("The lock is locked")

    channels := make([]chan int, 4)
    for i := 0; i < 4; i++ {
        channels[i] = make(chan int)
        go func(i int, c chan int) {
            fmt.Println("Not read lock: ", i)
            mutex.RLock()
            fmt.Println("Read Locked: ", i)
            fmt.Println("Unlock the read lock: ", i)
            time.Sleep(time.Second)
            mutex.RUnlock()
            c <- i
        }(i, channels[i])
    }
    time.Sleep(time.Second)
    fmt.Println("Unlock the lock")
    mutex.Unlock()
    time.Sleep(time.Second)

    for _, c := range channels {
        <-c
    }
}
```

程序输出：
```bash
Lock the lock
The lock is locked
Not read lock:  2
Not read lock:  3
Not read lock:  1
Not read lock:  0
Unlock the lock
Read Locked:  2
Read Locked:  1
Unlock the read lock:  2
Unlock the read lock:  1
Read Locked:  0
Read Locked:  3
Unlock the read lock:  0
Unlock the read lock:  3
```

### Unlock() 使用之前不存在 Lock()
```go
package main

import (
    "sync"
)

func main(){
    var rwmutex *sync.RWMutex
    rwmutex = new(sync.RWMutex)
    rwmutex.Unlock()
}
```
程序输出：
```bash
panic: sync: Unlock of unlocked RWMutex
```

## RWMutex 使用不当导致的死锁

### 示例1：
```go
package main

import (
    "sync"
)

func main(){
    var rwmutex *sync.RWMutex
    rwmutex = new(sync.RWMutex)
    rwmutex.Lock()
    rwmutex.Lock()
}
```
程序输出：
```bash
fatal error: all goroutines are asleep - deadlock!
```

### 示例2：
```go
package main

import (
    "sync"
)

func main(){
    var rwmutex *sync.RWMutex
    rwmutex = new(sync.RWMutex)
    rwmutex.Lock()
    rwmutex.RLock()
}
```

程序输出：
```bash
fatal error: all goroutines are asleep - deadlock!
```

### RUnlock() 之前不存在 RLock()
```go
package main

import (
    "sync"
)

func main(){
    var rwmutex *sync.RWMutex
    rwmutex = new(sync.RWMutex)
    rwmutex.RUnlock()
}
```
程序输出：
```bash
panic: sync: RUnlock of unlocked RWMutex
```

### RUnlock() 个数多于 RLock()
```go
package main

import (
    "sync"
)

func main(){
    var rwmutex *sync.RWMutex
    rwmutex = new(sync.RWMutex)
    rwmutex.RLock()
    rwmutex.RLock()
    rwmutex.RUnlock()
    rwmutex.RUnlock()
    rwmutex.RUnlock()
}
```

程序输出：
```bash
panic: sync: RUnlock of unlocked RWMutex
```