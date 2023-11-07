---
title: 理解Go Context机制
categories: Kubernetes
sage: false
date: 2020-04-03 15:34:48
tags: Go
---

## 1 什么是Context

最近在公司分析gRPC源码，proto文件生成的代码，接口函数第一个参数统一是ctx context.Context接口，公司不少同事都不了解这样设计的出发点是什么，其实我也不了解其背后的原理。今天趁着妮妲台风妹子正面登陆深圳，全市停工、停课、停业，在家休息找了一些资料研究把玩一把。

<!-- more -->

Context通常被译作上下文，它是一个比较抽象的概念。在公司技术讨论时也经常会提到上下文。一般理解为程序单元的一个运行状态、现场、快照，而翻译中上下又很好地诠释了其本质，上下上下则是存在上下层的传递，上会把内容传递给下。在Go语言中，程序单元也就指的是Goroutine。

每个Goroutine在执行之前，都要先知道程序当前的执行状态，通常将这些执行状态封装在一个Context变量中，传递给要执行的Goroutine中。上下文则几乎已经成为传递与请求同生存周期变量的标准方法。在网络编程下，当接收到一个网络请求Request，处理Request时，我们可能需要开启不同的Goroutine来获取数据与逻辑处理，即一个请求Request，会在多个Goroutine中处理。而这些Goroutine可能需要共享Request的一些信息；同时当Request被取消或者超时的时候，所有从这个Request创建的所有Goroutine也应该被结束。


## 2 context包

Go的设计者早考虑多个Goroutine共享数据，以及多Goroutine管理机制。Context介绍请参考Go Concurrency Patterns: Context，golang.org/x/net/context包就是这种机制的实现。

context包不仅实现了在程序单元之间共享状态变量的方法，同时能通过简单的方法，使我们在被调用程序单元的外部，通过设置ctx变量值，将过期或撤销这些信号传递给被调用的程序单元。在网络编程中，若存在A调用B的API, B再调用C的API，若A调用B取消，那也要取消B调用C，通过在A,B,C的API调用之间传递Context，以及判断其状态，就能解决此问题，这是为什么gRPC的接口中带上ctx context.Context参数的原因之一。

Go1.7(当前是RC2版本)已将原来的golang.org/x/net/context包挪入了标准库中，放在$GOROOT/src/context下面。标准库中net、net/http、os/exec都用到了context。同时为了考虑兼容，在原golang.org/x/net/context包下存在两个文件，go17.go是调用标准库的context包，而pre_go17.go则是之前的默认实现，其介绍请参考go程序包源码解读。

context包的核心就是Context接口，其定义如下：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}

```

>1. Deadline会返回一个超时时间，Goroutine获得了超时时间后，例如可以对某些io操作设定超时时间。

>2. Done方法返回一个信道（channel），当Context被撤销或过期时，该信道是关闭的，即它是一个表示Context是否已关闭的信号。

>3. 当Done信道关闭后，Err方法表明Context被撤的原因。

>4. Value可以让Goroutine共享一些数据，当然获得数据是协程安全的。但使用这些数据的时候要注意同步，比如返回了一个map，而这个map的读写则要加锁。

Context接口没有提供方法来设置其值和过期时间，也没有提供方法直接将其自身撤销。也就是说，Context不能改变和撤销其自身。那么该怎么通过Context传递改变后的状态呢？

## 3 context使用

无论是Goroutine，他们的创建和调用关系总是像层层调用进行的，就像人的辈分一样，而更靠顶部的Goroutine应有办法主动关闭其下属的Goroutine的执行（不然程序可能就失控了）。为了实现这种关系，Context结构也应该像一棵树，叶子节点须总是由根节点衍生出来的。

要创建Context树，第一步就是要得到根节点，context.Background函数的返回值就是根节点：

```go
func Background() Context
```

该函数返回空的Context，该Context一般由接收请求的第一个Goroutine创建，是与进入请求对应的Context根节点，它不能被取消、没有值、也没有过期时间。它常常作为处理Request的顶层context存在。

有了根节点，又该怎么创建其它的子节点，孙节点呢？context包为我们提供了多个函数来创建他们：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key interface{}, val interface{}) Context
```

函数都接收一个Context类型的参数parent，并返回一个Context类型的值，这样就层层创建出不同的节点。子节点是从复制父节点得到的，并且根据接收参数设定子节点的一些状态值，接着就可以将子节点传递给下层的Goroutine了。

再回到之前的问题：该怎么通过Context传递改变后的状态呢？使用Context的Goroutine无法取消某个操作，其实这也是符合常理的，因为这些Goroutine是被某个父Goroutine创建的，而理应只有父Goroutine可以取消操作。在父Goroutine中可以通过WithCancel方法获得一个cancel方法，从而获得cancel的权利。

第一个WithCancel函数，它是将父节点复制到子节点，并且还返回一个额外的CancelFunc函数类型变量，该函数类型的定义为：
```go
type CancelFunc func()
```

调用CancelFunc对象将撤销对应的Context对象，这就是主动撤销Context的方法。在父节点的Context所对应的环境中，通过WithCancel函数不仅可创建子节点的Context，同时也获得了该节点Context的控制权，一旦执行该函数，则该节点Context就结束了，则子节点需要类似如下代码来判断是否已结束，并退出该Goroutine：

```go
select {
    case <-cxt.Done():
        // do some clean...
}
```

WithDeadline函数的作用也差不多，它返回的Context类型值同样是parent的副本，但其过期时间由deadline和parent的过期时间共同决定。当parent的过期时间早于传入的deadline时间时，返回的过期时间应与parent相同。父节点过期时，其所有的子孙节点必须同时关闭；反之，返回的父节点的过期时间则为deadline。

WithTimeout函数与WithDeadline类似，只不过它传入的是从现在开始Context剩余的生命时长。他们都同样也都返回了所创建的子Context的控制权，一个CancelFunc类型的函数变量。

当顶层的Request请求函数结束后，我们就可以cancel掉某个context，从而层层Goroutine根据判断cxt.Done()来结束。

WithValue函数，它返回parent的一个副本，调用该副本的Value(key)方法将得到val。这样我们不光将根节点原有的值保留了，还在子孙节点中加入了新的值，注意若存在Key相同，则会被覆盖。

### 3.1 小结

context包通过构建树型关系的Context，来达到上一层Goroutine能对传递给下一层Goroutine的控制。对于处理一个Request请求操作，需要采用context来层层控制Goroutine，以及传递一些变量来共享。

>1. Context对象的生存周期一般仅为一个请求的处理周期。即针对一个请求创建一个Context变量（它为Context树结构的根）；在请求处理结束后，撤销此ctx变量，释放资源。

>2. 每次创建一个Goroutine，要么将原有的Context传递给Goroutine，要么创建一个子Context并传递给Goroutine。

>3. Context能灵活地存储不同类型、不同数目的值，并且使多个Goroutine安全地读写其中的值。

>4. 当通过父Context对象创建子Context对象时，可同时获得子Context的一个撤销函数，这样父Context对象的创建环境就获得了对子Context将要被传递到的Goroutine的撤销权。

5. 在子Context被传递到的goroutine中，应该对该子Context的Done信道（channel）进行监控，一旦该信道被关闭（即上层运行环境撤销了本goroutine的执行），应主动终止对当前请求信息的处理，释放资源并返回。

## 4 使用原则

Programs that use Contexts should follow these rules to keep interfaces consistent across packages and enable static analysis tools to check context propagation:
使用Context的程序包需要遵循如下的原则来满足接口的一致性以及便于静态分析。

>. Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it. The Context should be the first parameter, typically named ctx；不要把Context存在一个结构体当中，显式地传入函数。Context变量需要作为第一个参数使用，一般命名为ctx；

>. Do not pass a nil Context, even if a function permits it. Pass context.TODO if you are unsure about which Context to use；即使方法允许，也不要传入一个nil的Context，如果你不确定你要用什么Context的时候传一个context.TODO；

>. Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions；使用context的Value相关方法只应该用于在程序和接口中传递的和请求相关的元数据，不要用它来传递一些可选的参数；

>. The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines；同样的Context可以用来传递到不同的goroutine中，Context在多个goroutine中是安全的；

___

## Context 原理

Context 的调用应该是链式的，通过WithCancel，WithDeadline，WithTimeout或WithValue派生出新的 Context。当父 Context 被取消时，其派生的所有 Context 都将取消。

通过context.WithXXX都将返回新的 Context 和 CancelFunc。调用 CancelFunc 将取消子代，移除父代对子代的引用，并且停止所有定时器。未能调用 CancelFunc 将泄漏子代，直到父代被取消或定时器触发。go vet工具检查所有流程控制路径上使用 CancelFuncs。

## 遵循规则
遵循以下规则，以保持包之间的接口一致，并启用静态分析工具以检查上下文传播。

>1. 不要将 Contexts 放入结构体，相反context应该作为第一个参数传入，命名为ctx。 func DoSomething（ctx context.Context，arg Arg）error { // ... use ctx ... }
>2. 即使函数允许，也不要传入nil的 Context。如果不知道用哪种 Context，可以使用context.TODO()。
>3. 使用context的Value相关方法只应该用于在程序和接口中传递的和请求相关的元数据，不要用它来传递一些可选的参数
>4. 相同的 Context 可以传递给在不同的goroutine；Context 是并发安全的。

## Context 包
Context 结构体。
```go
// A Context carries a deadline, cancelation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

>1.  Done()，返回一个channel。当times out或者调用cancel方法时，将会close掉。
>2. Err()，返回一个错误。该context为什么被取消掉。
>3. Deadline()，返回截止时间和ok。
>4. Value()，返回值。

所有方法
```go
func Background() Context
func TODO() Context

func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```

上面可以看到Context是一个接口，想要使用就得实现其方法。在context包内部已经为我们实现好了两个空的Context，可以通过调用Background()和TODO()方法获取。一般的将它们作为Context的根，往下派生。

## WithCancel 例子

WithCancel 以一个新的 Done channel 返回一个父 Context 的拷贝。
```go
   229  func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
   230      c := newCancelCtx(parent)
   231      propagateCancel(parent, &c)
   232      return &c, func() { c.cancel(true, Canceled) }
   233  }
   234  
   235  // newCancelCtx returns an initialized cancelCtx.
   236  func newCancelCtx(parent Context) cancelCtx {
   237      return cancelCtx{
   238          Context: parent,
   239          done:    make(chan struct{}),
   240      }
   241  }
```

此示例演示使用一个可取消的上下文，以防止 goroutine 泄漏。示例函数结束时，defer 调用 cancel 方法，gen goroutine 将返回而不泄漏。
```go
package main

import (
    "context"
    "fmt"
)

func main() {
    // gen generates integers in a separate goroutine and
    // sends them to the returned channel.
    // The callers of gen need to cancel the context once
    // they are done consuming generated integers not to leak
    // the internal goroutine started by gen.
    gen := func(ctx context.Context) <-chan int {
        dst := make(chan int)
        n := 1
        go func() {
            for {
                select {
                case <-ctx.Done():
                    return // returning not to leak the goroutine
                case dst <- n:
                    n++
                }
            }
        }()
        return dst
    }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel() // cancel when we are finished consuming integers

    for n := range gen(ctx) {
        fmt.Println(n)
        if n == 5 {
            break
        }
    }
}
```

WithDeadline 例子
```go
   369  func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) {
   370      if cur, ok := parent.Deadline(); ok && cur.Before(deadline) {
   371          // The current deadline is already sooner than the new one.
   372          return WithCancel(parent)
   373      }
   374      c := &timerCtx{
   375          cancelCtx: newCancelCtx(parent),
   376          deadline:  deadline,
   377      }
   ......
```

可以清晰的看到，当派生出的子 Context 的deadline在父Context之后，直接返回了一个父Context的拷贝。故语义上等效为父。

WithDeadline 的最后期限调整为不晚于 d 返回父上下文的副本。如果父母的截止日期已经早于 d，WithDeadline （父，d） 是在语义上等效为父。返回的上下文完成的通道关闭的最后期限期满后，返回的取消函数调用时，或当父上下文完成的通道关闭，以先发生者为准。

看看官方例子：
```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    d := time.Now().Add(50 * time.Millisecond)
    ctx, cancel := context.WithDeadline(context.Background(), d)

    // Even though ctx will be expired, it is good practice to call its
    // cancelation function in any case. Failure to do so may keep the
    // context and its parent alive longer than necessary.
    defer cancel()

    select {
    case <-time.After(1 * time.Second):
        fmt.Println("overslept")
    case <-ctx.Done():
        fmt.Println(ctx.Err())
    }
}
```

WithTimeout 例子
WithTimeout 返回 WithDeadline(parent, time.Now().Add(timeout))。
```go
   436  func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
   437      return WithDeadline(parent, time.Now().Add(timeout))
   438  }
```

看看官方例子：
```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // Pass a context with a timeout to tell a blocking function that it
    // should abandon its work after the timeout elapses.
    ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
    defer cancel()

    select {
    case <-time.After(1 * time.Second):
        fmt.Println("overslept")
    case <-ctx.Done():
        fmt.Println(ctx.Err()) // prints "context deadline exceeded"
    }
}
```

WithValue 例子
```go
   454  func WithValue(parent Context, key, val interface{}) Context {
   454      if key == nil {
   455          panic("nil key")
   456      }
   457      if !reflect.TypeOf(key).Comparable() {
   458          panic("key is not comparable")
   459      }
   460      return &valueCtx{parent, key, val}
   461  }
```

WithValue 返回的父与键关联的值在 val 的副本。

使用上下文值仅为过渡进程和 Api 的请求范围的数据，而不是将可选参数传递给函数。

提供的键必须是可比性和应该不是字符串类型或任何其他内置的类型以避免包使用的上下文之间的碰撞。WithValue 用户应该定义自己的键的类型。为了避免分配分配给接口 {} 时，上下文键经常有具体类型结构 {}。另外，导出的上下文关键变量静态类型应该是一个指针或接口。

看看官方例子：
```go
package main

import (
    "context"
    "fmt"
)

func main() {
    type favContextKey string

    f := func(ctx context.Context, k favContextKey) {
        if v := ctx.Value(k); v != nil {
            fmt.Println("found value:", v)
            return
        }
        fmt.Println("key not found:", k)
    }

    k := favContextKey("language")
    ctx := context.WithValue(context.Background(), k, "Go")

    f(ctx, k)
    f(ctx, favContextKey("color"))
}
```

参考：
[1] https://blog.golang.org/context
[2] http://blog.golang.org/pipelines
[3] http://studygolang.com/articles/5131
[4] http://blog.csdn.net/sryan/article/details/51969129
[5] https://peter.bourgon.org/blog/2016/07/11/context.html
[6] http://www.tuicool.com/articles/vaieAbQ