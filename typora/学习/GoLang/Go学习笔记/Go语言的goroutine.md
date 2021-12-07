# Goroutine

I/O操作的时候，是将任务交给DMA来处理，请求发出后CPU就不管了，在DMA处理完后通过中断通知CPU处理完成了。I/O操作消耗的cpu时间很少。

IO相关：

https://segmentfault.com/a/1190000003063859

https://www.cnblogs.com/mao3714/articles/8184262.html

GPM:

https://www.cnblogs.com/linguanh/p/9510746.html

协程：

https://www.cnblogs.com/kazetotori/p/6896653.html

### 并发模型

- 多线程/多进程
- IO多路复用+异步回调
  - 多路复用:select/poll/epoll
  - 异步IO: 内核事件通知
- Goroutine

### Goroutine

- 调度模型 GMP(S)
  - M:M代表内核级线程，一个M就是一个线程，goroutine就是跑在M之上的；M是一个很大的结构，里面维护小对象内存cache（mcache）、当前执行的goroutine、随机数发生器等等非常多的信息
  - G:代表一个goroutine，它有自己的栈，instruction pointer和其他信息（正在等待的channel等等），用于调度。
  - P:P全称是Processor，处理器，它的主要用途就是用来执行goroutine的，所以它也维护了一个goroutine队列，里面存储了所有需要它来执行的goroutine
  - Sched：代表调度器，它维护有存储M和G的队列以及调度器的一些状态信息等。
- 基本使用
  - 创建单个Goroutine
  - 创建多个Goroutine
- 死锁和友好退出
  - 非缓冲通道上如果发生了流入无流出，或者流出无流入，就会引起死锁。
  - 把没取走的取走
  - 创建带缓冲的Channel
- 同步Goroutine
  - WaitGroup(Add/Done/Wait)
  - Channel
- Goroutine间通信
  - Channel
  - 全局变量

### Channel

同步模式的关键是找到匹配的接收或发送方，找到则直接拷贝数据，找不到就将自身打包 后放入等待队列，由另一方复制数据并唤醒。

在同步模式下，channel 的作用仅是维护发送和接收者队列，数据复制与 channel 无关。 另外在唤醒后，需要验证唤醒者身份，以此决定是否有实际的数据传递。

异步模式围绕缓冲槽进行。当有空位时，发送者向槽中复制数据;有数据后，接收者从槽 中获取数据。双方都有唤醒排队另一方继续工作的责任。

关闭操作将所有排队者唤醒，并通过 chan.closed、g.param 参数告知由 close 发出。

- 向 closed channel 发送数据，触发 panic。
- 从 closed channel 读取数据，返回零值。
- 无论收发，nil channel 都会阻塞。

除用 range 外，还可用 ok-idiom 模式判断 channel 是否关闭。

```
for {
    if d, ok := <-data; ok {
        fmt.Println(d)
    } else {
        break
    }
}
```

- 介绍
  - 俗称管道，用于数据传递或数据共享，其本质是一个先进先出的队列
  - 使用goroutine+channel进行数据通讯简单高效，同时也线程安全，多个goroutine可同时修改一个channel，不需要加锁。
- 基本使用
  - 管道如果未关闭，在读取超时会则会引发deadlock异常
  - 管道如果关闭进行写入数据会pannic
  - 当管道中没有数据时候再行读取或读取到默认值，如int类型默认值是0
  - 使用range循环管道，如果管道未关闭会引发deadlock错误。
  - 如果采用for死循环已经关闭的管道，当管道没有数据时候，读取的数据会是管道的默认值，并且循环不会退出。
- 带缓冲的Channel
  - 带缓冲区channel：定义声明时候制定了缓冲区大小(长度)，可以保存多个数据。
  - 不带缓冲区channel：只能存一个数据，并且只有当该数据被取出时候才能存下一个数据。
- 阻塞与非阻塞
- 只读与只写

### Select

- 多路复用
- 基本使用
  - select-case实现非阻塞channel
  - select中的某个case时候，那么该case返回，若都不满足case，则走default分支。
  - 如果有多个case都可以运行，select会随机公平地选出一个执行，其他不会执行
  - case后面必须是channel操作，否则报错。
  - select中的default子句总是可运行的。所以没有default的select才会阻塞等待事件
  - 没有运行的case，那么阻塞事件发生死锁
- 实战
  - Timeout机制(超时判断)
  - 判断channel是否阻塞(或者说channel是否已经满了)
  - 退出机制