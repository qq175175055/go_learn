# 并发：go多线程

[toc]

并发编程含义广泛，这里特指多线程编程。go并发通过goroutine特性完成，线程由操作系统调度完成。  

## 1. 轻量级线程：goroutine

goroutine是轻量级线程，可以根据需要随时创建。goroutine由go程序运行时调度和管理，go程序会智能的将goroutine的任务合理分配给每个CPU。go程序从main()函数开始，在程序启动时就为main()函数创建一个默认的goroutine。  

### 使用普通函数创建goroutine

```go
go 函数名(参数列表) // 注意：函数返回值会被忽略
```

```go
package main

import (
    "fmt"
    "time"
)

func running() {
    var times int
    for {
        times++
        fmt.Println("tick:", times)
        time.Sleep(time.Second)
    }
}

func main() {
    go running()

    var input string
    fmt.Scanln(&input)
}
```

### 使用匿名函数创建goroutine

```go
go func(参数列表) {
    函数体
}(实际调用参数列表)
```

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    go func() {
        var times int
        for {
            times++
            fmt.Println("tick:", times)
            time.Sleep(time.Second)
        }
    }()

    var input string
    fmt.Scanln(&input)
}
```

可以通过下述函数调整并发的cpu：

```go
runtime.GOMAXPROCS(逻辑CPU数量)

runtime.GOMAXPROCS(runtime.NumCPU())
```

## 2. 管道（channel）

单纯函数并发是没有意义的，函数之间的数据交换往往才有意义，而管道是多个goroutine间通信的通道。  

管道是一种特殊的类型，在任意时刻，只能有一个goroutine访问管道进行发送或获取数据，goroutine之间通过管道就可以通信。  

声明管道：

```go
var 管道变量 chan 管道内数据类型
声明后的数据管道是nil，需要通过make生成实例，如：
管道实例 := make(chan 管道内数据类型)

ch1 := make(chan int)
```

### 2.1 发送数据

```go
管道的发送使用特殊的操作符 <-，将数据通过管道发送的格式：
管道变量 <- 值 // 值可以是常量/变量/表达式/函数返回值等
```

将数据通过管道发送，如果接收方一直没有接收，则发送操作将会持续阻塞。go会智能发现持续阻塞的语句并给出提示。  

### 2.2 接收数据

```go
管道的接收同样使用操作符 <-
```

管道接收特点：

* 管道的收发操作在不同的goroutine间进行
* 接收将持续阻塞直到发送方发送数据
* 每次接收一个元素

```go
// 阻塞接收数据
data := <-ch

// 非阻塞接收数据
data, ok := <-ch

// 接收数据并忽略
<-ch

// 循环接收
for data := range ch {
    // do something
}
```

### 2.3 单向管道

go的管道可以通过声明时约束操作方向，如只发送或者接收。这种被约束方向的管道被称作单向管道。  

```go
// 单向管道的声明
var 管道实例 chan<- 管道内数据类型 // 只能发送管道
var 管道实例 <-chan 管道内数据类型 // 只能接收管道

ch := make(chan, int)
var chSendOnly chan<- int = ch
var chRecvOnly <-chan int = ch
```

### 2.4 带缓冲的管道

为管道增加一个有限大小的空间形成缓冲管道，缓冲管道在缓冲中有可用空间时发送数据并不会造成阻塞，同理，缓冲管道在缓冲中有数据时接收数据并不会造成阻塞。  

```go
// 创建带缓冲的管道
管道实例 := make(chan 管道内数据类型, 缓冲大小)

ch := make(chan int, 3)
```

### 2.5 管道的多路复用

多路复用通常表示在一个信道上传输多路数据的过程和技术。  
go提供select关键字，可以同时响应多个管道的操作，select的每个case都会对应一个管道的收发过程。当收发完成时就会触发case中程序的语句。多个操作在每次select中挑选一个进行响应。格式如下：  

```go
select {
    case 操作1:
        响应操作1
    case 操作2:
        响应操作2
    // more case
    default:
        没有操作情况
}

// 操作N的语句示例：
// case <-ch:
// case data := <-ch:
// case ch <- 100:
```

管道的例子，模拟远程过程调用（RPC）。  

```go
// 客户端请求和接收
func RPCClient(ch chan string, req string) (string, error) {
    ch <- req

    select {
        case ack := <-ch:
            return ack, nil
        case <-time.After(time.Second):
            return "", errors.New("time out")
    }
}

// 服务器接收和反馈
func RPCServer(ch chan string) {
    for {
        data := <-ch
        fmt.Println("server recv: ", data)
        ch <- "roger"
    }
}

// main
func main() {
    ch := make(chan string)
    go RPCServer(ch)

    recv, err := RPCClient(ch, "hi")
    if err != nil {
        fmt.Println(err)
    } else {
        fmt.Println("client received: ", recv)
    }
}
```

## 2.6 同步

go除了使用goroutine保证数据同步之外，还有其他的方式。如原子访问包、互斥锁、以及等待组。  

原子访问包例子：

```go
package main

import (
    "fmt"
    "sync/atomic"
)

var seq int64

func GetID() int64 {
    return atomic.AddInt64(&seq, 1)
}

func main() {
    for i := 0; i < 10; i++ {
        go GetID()
    }
    fmt.Println(GetID())
}
```

互斥锁的例子：

```go
package main

import (
    "fmt"
    "sync"
)

var (
    count int
    countGuard sync.Mutex
)

func GetCount() int {
    countGuard.Lock()
    defer countGuard.Unlock()
    return count
}

func SetCount() {
    countGuard.Lock()
    count = c
    countGuard.Unlock()
}

func main() {
    SetCount(1)
    fmt.Println(GetCount())
}

// 当然，除了sync.Mutex，还有读写互斥锁sync.RWMutex
```

等待组的例子：

```go
package main

import (
    "fmt"
    "net/http"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    var urls = []string{
        "http://github.com"
        "http://www.baidu.com"
        "http://www.golangtc.com"
    }

    for _, url := range urls {
        wg.Add(1)
        go func(url string) {
            defer wg.Done()
            _, err := http.Get(url)
            fmt.Println(url, err)
        }(url)
    }

    wg.Wait()

    fmt.Println("over")
}
```
