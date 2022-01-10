---
title: go design (五) channel
date: 2022-01-10 19:07:10
categories: golang 
tags:
- channel
- 《go语言设计与实现》
---

# go channel 的设计与实现

`golang` 中推崇的金句就是 **不要通过共享内存来通信，要通过通信的方式来共享内存**,其通信的载体就是 `channel` , `golang` 特有的关键字（数据结构），在 `golang` 中要实现并发编程成本很低， 一个 `go` 关键词 就可以启动一个 `goroutine` ，那么多个 `goroutine` 之间的数据传输该怎么处理呢？就有了 `channel` 通道，这种数据类型 来帮助在 多个 `goroutine` 进行信息传输。

## 从实例开始 channel 介绍之旅

### 创建与使用

#### 缓存与不带缓存的 channel

```go
c := make(chan int) // 不带缓存的 channel
c := make(chan int, 0) // 不带缓存的 channel
c := make(chan int, 2) // 带缓存的 channel
var s chan int // 特殊 channel nil

fmt.Println(c, s) // 0xc000086060 <nil>
```

- `make` 出来的 `chan` 为 实际结构地址的引用， 而 声明的 `channel 为 nil 则永久性的读写堵塞, 且不能被 close 不然会 panic`
- 不带缓存的 `channel`
  - 进行读操作的时候，如果无数据 会进入堵塞状态，直到协程内有数据被写入
  - 进行写操作的时候，如果无协程在读取数据，也会进入堵塞状态，直到数据被读取
- 带缓存 的 `channel`
  - 进行写操作的时候，如果缓存还有空间则不会被堵塞，否则也会堵塞
  - 进行读操作的时候，如果无数据会进入堵塞状态，直到协程内有数据被写入

```go
//堵塞 
func main() {
	c := make(chan int) // 不带缓存的 channel

	go func() {
		time.Sleep(1 * time.Second)
		ok := <-c
		fmt.Println(ok)
	}()

	c <- 1
	time.Sleep(1 * time.Second)
}
```

```go
//带缓冲 
func main() {
	c := make(chan int,2) // 带缓存的 channel

	go func() {
		for{
			time.Sleep(1 * time.Second)
			ok := <-c
			fmt.Println(ok)
		}
	}()

	c <- 1
	c <- 2
	time.Sleep(3 * time.Second)
}
```

#### channel 的两个属性

```go
c := make(chan int, 2) // 带缓存的 channel
fmt.Println(len(c), cap(c))// 0  2 
```

- `len()` 为当前 `channel` 已经使用的缓存量
- `cap()` 为 当前 `channel` 最大的缓存量

####  select 为多 `channel` 处理而生

在上述的使用中 如果在 一个协程中 使用多个 channel`，如果一个 `channel` 堵塞 ，那么代码就没法 执行到 下一个 `channel` ，从而导致运行时 死锁

比如：

```go
c := make(chan int) // 不带缓存的 channel
d := make(chan int) // 不带缓存的 channel
go func() {
    for {
        ok := <-c
        ok2 := <-d
        fmt.Println(ok,ok2)
    }
}()
d <- 1
c <- 1
time.Sleep(3 * time.Second)
//fatal error: all goroutines are asleep - deadlock!
```

这种情况下如果不知道 `c,d` 谁会先发送数据的情况 就会 直接报错，相互死锁，程序中断。

- 通过 select 来对多个channel 进行收发控制 应运而生

- select 的特性

  - `select` 无 `case` 属性时，会直接堵塞代码执行，切记勿在主协程中使用，不然直接死锁
  - `select` 只有一个 `case channel` 时，会一直堵塞，直到有数据进入

  ```go
  c := make(chan int) // 不带缓存的 channel
  go func() {
      select {
          case ok := <-c :
          fmt.Println(ok)
      }
  }()
  c <- 1
  time.Sleep(1 * time.Second)
  ```

  - `select` 能在 `channel` 上进行非阻塞的收发操作；如果 存在 `case default` 无数据直接进入 `default case`

  ```go
  c := make(chan int) // 不带缓存的 channel
  go func() {
  	select {
  	case ok := <-c:
  		fmt.Println(ok)
  	default:
  		fmt.Println("未接受到数据")
  	}
  }()
  
  c <- 1
  time.Sleep(1 * time.Second)
  ```

  - select 在遇到多个 channel 同时响应时，会随机执行一种情况； 这里用 同一个 `channel` 不同 `channel` 的同一时刻，效果也是类似，只会有一个分支被读取与执行

  ```go
  c := make(chan int) // 不带缓存的 channel
  go func() {
  	select {
  	case ok := <-c:
  		fmt.Println("case 1", ok)
  	case ok := <-c:
  		fmt.Println("case 2", ok)
  	case ok := <-c:
  		fmt.Println("case 3", ok)
  	}
  }()
  
  c <- 1
  time.Sleep(1 * time.Second)
  // 多次响应 的结果不一致
  ```

#### `for range` 与 `channel` 的配合使用

    ```go
    func main() {
        c := make(chan int, 2) // 带缓存的 channel
    
        go func() {
            for val := range c {
                fmt.Println(val)
            }
        }()
    
        for i := 0; i < 10; i++ {
            c <- i
            if i == 5 {
                close(c)
                break
            }
        }
    
        time.Sleep(time.Second * 1)
    }
    ```

- 对于 `for range` 进行迭代 的 `channel` 除非 `channel` 被关闭，不然会一直堵塞下去
- 对于 关闭后 `channel` 内还存在未处理完的数据情况，也会被读取出来


#### `close()` 函数与 `channel` 的关闭

- `close()` 只是相对于 发送方的概念，`close` 之后的 `channel` 不能进行发送操作，但是从 `channel `中读取数据的行为是允许的，存在数据时，拿到的是数据，不存在时 ，拿到的是 零值，且不会堵塞

- 但是 `close` 函数不能针对与一个 `channel` 执行多次,  不然 就会 出现 `panic: close of closed channel` 的运行时错误

- `close` 之后的 `channel` ，再进行发送会出现 `panic: send on closed channel`

  ```go
  func main() {
      c := make(chan int) // 带缓存的 channel
  
      go func() {
          for val := range c {
              fmt.Println(val)
          }
  
          value, ok := <-c
          fmt.Println(value, ok) // 0 false
      }()
  
      c <- 0
      c <- 1
      close(c)
      time.Sleep(time.Second * 2)
  }
  ```

#### 用法-超时控制

- 使用 `time.After()` 来实现 延时的 `channel` 信息发送

  ```go
  func main() {
      c := make(chan int) // 不带缓存的 channel
  
      go func() {
          select {
          case ok := <-c:
              fmt.Println(ok)
          case <-time.After(time.Second * 1):
              fmt.Println("一秒后超时")
          }
      }()
      time.Sleep(2 * time.Second)
  }
  ```

#### 用法-控制并发执行协程数量（协程池）

- 使用

  ```go
  func main() {
  
      limit := make(chan int, 3) // 控制同时执行的 goroutine 个数
  
      work := [100]int{}
  
      for k := range work {
          go func(k int) {
              limit <- 1
               w(k)
              <-limit
          }(k)
      }
  
      time.Sleep(time.Second * 20)
  }
  
  func w(i int) {
      time.Sleep(time.Second * 1)
      fmt.Println("deal task", i)
  }
  ```

#### 用法-生产消费模型

- 用于解耦合

  ```go
  func main() {
      s := make(chan int)
  
      // 消费
      go func() {
          for v := range s {
              go work(v)
          }
      }()
  
      // 生产
      for i := 0; i < 10; i++ {
          s <- i
      }
  
      time.Sleep(time.Second * 20)
  }
  
  func work(taskId int) {
      time.Sleep(time.Second * 1)
      fmt.Println("deal task ", taskId)
  }
  
  ```

## golang 中的 设计与实现

到目前为止，我们把 `channel` 的基础功能 过了一遍, 感觉上确实是挺复杂与强大的，接下来会去看看 `channel` 的数据结构，跟写入，读取的流程，通过这些流程，更好的理解为什么，`channel` 会有这样的特性.

### channel 实现的数据结构

- 源码中的定义

```go
// runtime/chan.go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}

type waitq struct {
	first *sudog
	last  *sudog
}

type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g

	next *sudog
	prev *sudog
	elem unsafe.Pointer // data element (may point to stack)

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool

	// success indicates whether communication over channel c
	// succeeded. It is true if the goroutine was awoken because a
	// value was delivered over channel c, and false if awoken
	// because c was closed.
	success bool

	parent   *sudog // semaRoot binary tree
	waitlink *sudog // g.waiting list or semaRoot
	waittail *sudog // semaRoot
	c        *hchan // channel
}
```

- 可以看出来底层的循环队列主要由 `qcount` `dataqsiz`、`buf`、`sendx`、`recvx` 构建
  - `qcount`    `channel` 的总长度
  - `dataqsiz` 循环队列的长度
  - `buf`         缓冲区数据指针
  - `sendx`      发送操作处理到的位置
  - `recvx`      接收操作处理到的位置

- `recvq` `sendq` 代表 等待中的 `goroutine` 链表
- `runtime.makechan()` 为创建 `channel` 的函数，缓存与非缓存 一个是底层分配空间大小的区别，二是对应的 `elemsize qcount dataqsiz` 属性值不同

### channel 的发送

- 发送的三种情况
  - 如果当前 `channel` 的 `recvq` 上存在已经被阻塞的 `goroutine`，那么会直接将数据发送给当前 `goroutine` 并将其设置成 可运行状态；
  - 如果 `channel` 存在缓冲区并且其中还有空闲的容量，我们会直接将数据存储到缓冲区 `sendx` 所在的位置上；
  - 如果不满足上面的两种情况，会创建一个 `runtime.sudog` 结构并将其加入 `channel` 的 `sendq` list中，当前 `goroutine` 也会**陷入阻塞**等待其他的协程从 `channel` 接收数据；
- 发送数据时两个会触发 `goroutine` 调度的时机：
  - `recvq` 上存在已经被阻塞的 `goroutine`,立刻设置处理器的 `runnext` 属性，但是并不会立刻触发调度；
  - 找到接收方并且缓冲区已经满了，这时会将自己加入 `channel` 的 `sendq` 队列并调用 `runtime.goparkunlock` 触发 `goroutine` 的调度让出处理器的使用权；

### channel 的接受

- 接受主要就是 调用  `runtime.chanrecv` 

- 接受的几种情况

  - 从一个 `空（nil) channel` 接收数据时会直接调用 `runtime.gopark` 让出处理器的使用权

  - `channel` 被关闭, 且缓冲区中不存在任何数据，那么会清除数据并立刻返回。

  - channel 正常时

    - `sendq` 队列中存在挂起的 `goroutine`，会将 `recvx` 索引所在的数据拷贝到接收变量所在的内存空间上，并将 `sendq` 队列中 `goroutine` 的数据拷贝到缓冲区；

    - 缓冲区中存在数据，直接读取 `recvx` 索引对应的数据；
    - 默认情况下会挂起当前的 `goroutine`，将 `runtime.sudog` 结构加入`recvq` 队列并陷入休眠等待调度器的唤醒；

- 调度时机

  - 当 channel 为空(nil) 时；
  - 当缓冲区中不存在数据并且也不存在数据的发送者时；

### close

- `close` 函数先上一把大锁，接着把所有挂在这个 channel 上的 `recvq` 和 `sendq` 全都连成一个 `sudog` 链表，再解锁。最后，再将所有的 `sudog` 全都唤醒
- 所以 当 存在有 `sendq` 被唤醒而 `chan` 本身被关闭时候，会直接被 `panic`