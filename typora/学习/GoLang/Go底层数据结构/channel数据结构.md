一、channel数据结构

![img](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/20/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zNTY1NTY2MQ==,size_16,color_FFFFFF,t_70.png)

chan 数据结构：src/runtime/channel;

主要构成：

- 环状队列（缓冲区）；
- 等待队列（读队列和写队列）；
- 元素信息（类型、大小）；

1. 环状队列（缓冲区）

- buf：指向队列的内存；
- dataqsiz：队列长度，可缓冲的元素个数；
- qcount：缓冲队列中中的元素个数；
- sendx：下次写入数据的位置；
- recvx：下次读取数据的位置；

2. 等待队列
阻塞
从channel读数据，如果channel缓冲区为空或者没有缓冲区，当前goroutine会被阻塞（等待读消息的 goroutine队列）；

向channel写数据，如果channel缓冲区已满或者没有缓冲区，当前goroutine会被阻塞（等待写消息的goroutine队列）；

唤醒
因读阻塞的goroutine，会被向channel写入数据的goroutine唤醒；

因写阻塞的goroutine，会被从channel读数据的goroutine唤醒；



3. 类型信息
elemtype代表类型，用于数据传递过程中的赋值；
elemsize代表类型大小，用于在buf中定位元素位置；

4. 锁
一个channel同时仅允许被一个goroutine读写；

二、channel读写
1. 创建channel
    //创建channel的伪代码
    func makechan(t *chantype, size int) *hchan {
        //1.创建channle
        var c *hchan
        c = new(hchan)
        //2.元素类型
        c.elemsize = 元素类型大小
        c.elemtype = 元素类型
        //3.环状缓冲队列
        c.buf = malloc(元素类型大小*size)
        c.dataqsiz = size
        return c
    }
2. 向channel中写数据
1. 如果等待接收队列recvq不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从recvq取出G,并把数据写入，最后把该G唤醒，结束发送过程；（等待接收队列recvq）

2.如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程；（缓冲队列buf）

3.如果缓冲区中没有空余位置，将待发送数据写入G，将当前G加入sendq，进入睡眠，等待被读goroutine唤醒；（等待发送队列sendq）



3. 从channel读数据
1.如果等待发送队列sendq不为空，且没有缓冲区，直接从sendq中取出G，把G中数据读出，最后把G唤醒，结束读取过程；（等待发送独立sendq）

2.如果等待发送队列sendq不为空，且有缓冲区，此时说明缓冲区已满，从缓冲区中首部读出数据，把G中数据写入缓冲区尾部，把G唤醒，结束读取过程；（缓冲队列sendq）

3.如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程；（缓冲队列sendq）

4.将当前goroutine加入recvq，进入睡眠，等待被写goroutine唤醒；（等待接收对了recvq）



4. 关闭channel
关闭channel时，会把recvq中的G全部唤醒，本该写入G的数据位置为nil。

关闭channel时，把sendq中的G全部唤醒，但这些G会panic。

panic出现的常见场景还有：

1）关闭值为nil的channel；
    
2）关闭已经被关闭的channel（多次唤醒）；
    
3）向已经关闭的channel写数据；

三、常见用法
1. 单向channel
单向channel指只能用于发送或接收数据，实际上也没有单向channel。

所谓单向channel只是对channel的一种使用限制，这跟C语言使用const修饰函数参数为只读是一个道理。

    func readChan(chanName <-chan int)： //通过形参限定函数内部只能从channel中读取数据
    func writeChan(chanName chan<- int)： //通过形参限定函数内部只能向channel中写入数据
2. select
使用select可以监控多channel;

比如监控多个channel，当其中某一个channel有数据时，就从其读出数据;

        select {
        //chan1有数据，走case1分支
        case e := <- chan1 :
            fmt.Printf("Get element from chan1: %d\n", e)
        //chan2有数据，走case2分支
        case e := <- chan2 :
            fmt.Printf("Get element from chan2: %d\n", e)
        //chan1和chan2都没有数据，或其他chan有数据，走default分支
        //若chan1和chan2都没有数据，或其他chan有数据，且没有default分支，阻塞
        default:
            fmt.Printf("No element in chan1 and chan2.\n")
            time.Sleep(1 * time.Second)
        }
3. range
通过range可以持续从channel中读出数据，好像在遍历一个数组一样;

当channel中没有数据时会阻塞当前goroutine，与读channel时阻塞处理机制一样。
