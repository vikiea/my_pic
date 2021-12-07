## 什么是内存模型

首先内存模型并不是指arena/spans/bitmap（如下图）。这些是内存划分。



![img](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/13/612f70f026cb0af3533cd6f537fb4e37.png)

为了保证共享内存的正确性（可见性、有序性、原子性），内存模型定义了共享内存系统中多线程程序读写操作行为的规范。

通过这些规则来规范对内存的读写操作，从而保证指令执行的正确性。它与处理器有关、与缓存有关、与并发有关、与编译器也有关。

它解决了 CPU 多级缓存、指令重排等导致的内存访问问题，保证了并发场景下的一致性、原子性和有序性。

上面提到，内存模型与处理器有关、与缓存有关、与并发有关、与编译器也有关，那么我们在编写Go程序的时候，需要去了解CPU等底层特性吗？其实是不需要的！因为内存模型是抽象的，在不同的平台下，编译器会生成合适的内存屏障，帮我们屏蔽了底层的差异。这里将面向抽象编程的思想体现的淋漓尽致！

## Golang的内存模型

Go 中也定义了Happens Before以及各种发生Happens Before关系的操作，因为有了这些Happens Before操作的保证，我们写的多goroutine的程序才会按照我们期望的方式来工作

### 什么是Happens Before

如果A happens before B，那么A的执行结果对B可见（**并不一定表示A比B先执行，如果A与B执行的顺序对结果没有影响是可以重排序的**）

### Go 中定义的Happens Before保证

#### 单线程

在单线程环境下，所有的表达式，按照代码中的先后顺序，具有Happens Before关系
——**说白了，就是能够保证不管CPU，编译器怎么优化，代码从结果看按顺序执行的**。
举个例子，如下代码中如果CPU或者编译器将指令重排（处于优化目的）后，有可能是E1->E3->E2的顺序执行，那么结果就会不对。Happens Before关系杜绝了这种错误。

```
package main

import "fmt"

func main() {
    a := 1//E1
    a++//E1
    fmt.Print(a + 1)//E3
}
```

#### Init 函数

- 如果包P1中导入了包P2，则P2中的init函数Happens Before 所有P1中的操作
- main函数Happens After 所有的init函数

——说白了，就是保证从结果看，Go程序的启动顺序如下

1. 按顺序导入所有被 main 包引用的其它包，然后在每个包中执行如下流程：
2. 如果该包又导入了其它的包，则从第一步开始递归执行，但是每个包只会被导入一次。
3. 然后以相反的顺序在每个包中初始化常量和变量，如果该包含有 init 函数的话，则调用该函数。
4. 在完成这一切之后，main 也执行同样的过程，最后调用 main 函数开始执行程序。

#### Goroutine

- Goroutine的创建Happens Before所有此Goroutine中的操作
- Goroutine的销毁Happens After所有此Goroutine中的操作

——说白了，就是保证了Goroutine创建前修改的数据在Goroutine执行时一定已经生效，以及Goroutine执行时的修改在Goroutine销毁后主Goroutine再去读取时一定已经生效

#### Channel

- 对一个元素的send操作Happens Before对应的**receive完成**操作
  ——**说白了，就是保证了receive操作，在接受完成之前一定会阻塞，所以我们可以使用channel做同步**
- 对channel的close操作Happens Before receive 端的收到关闭通知操作
  ——**说白了，就是保证send端close通道完成之后，receive 端才会收到关闭通知**
- 对于Unbuffered Channel，对一个元素的receive 操作Happens Before对应的**send完成**操作
  ——**说白了，就是保证元素send了没被receive时，在send端会阻塞**
- 对于Buffered Channel，假设Channel 的buffer 大小为C，那么对第k个元素的receive操作，Happens Before第k+C个**send完成**操作。可以看出上一条Unbuffered Channel规则就是这条规则C=0时的特例
  ——**说白了，就是保证了在Buffer满了之后，元素send了没被receive时，在send端会阻塞**

注意这里面，send和send完成，这是两个事件，receive和receive完成也是两个事件。

然后，Buffered Channel这里有个坑，它的Happens Before保证比UnBuffered 弱，这个弱只在【在receive之前写，在send之后读】这种情况下有问题。而【在send之前写，在receive之后读】，这样用是没问题的，这也是我们通常写程序常用的模式，千万注意这里不要弄错！

#### Lock

Go里面有Mutex和RWMutex两种锁，RWMutex除了支持互斥的Lock/Unlock，还支持共享的RLock/RUnlock。

- 对于一个Mutex/RWMutex，设n < m，则第n个Unlock操作Happens Before第m个Lock操作。
- 对于一个RWMutex，存在数值n，RLock操作Happens After 第n个UnLock，其对应的RUnLockHappens Before 第n+1个Lock操作。

简单理解就是这一次的Lock总是Happens After上一次的Unlock，读写锁的RLock HappensAfter上一次的UnLock，其对应的RUnlock Happens Before 下一次的Lock。

#### Once

once.Do中执行的操作，Happens Before 任何一个once.Do调用的返回