# Go语言的反射机制

## 解读反射的三大定律
Go 语言里有个反射三定律，是你在学习反射时，很重要的参考：
```
Reflection goes from interface value to reflection object.

Reflection goes from reflection object to interface value.

To modify a reflection object, the value must be settable.
```

翻译一下，就是：

- **反射可以将接口类型变量 转换为“反射类型对象”；**

- **反射可以将 “反射类型对象”转换为 接口类型变量；**

- **如果要修改 “反射类型对象” 其类型必须是 可写的；**

### 第一定律
> Reflection goes from interface value to reflection object.

为了实现从接口变量到反射对象的转换，需要提到 reflect 包里很重要的两个方法：

reflect.TypeOf(i) ：获得接口值的类型

reflect.ValueOf(i)：获得接口值的值

这两个方法返回的对象，我们称之为反射对象：Type object 和 Value object。

![](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/14/16287334356189.jpg)


举个例子，看下这两个方法是如何使用的？
```Go
package main

import (
"fmt"
"reflect"
)

func main() {
    var age interface{} = 25

    fmt.Printf("原始接口变量的类型为 %T，值为 %v \n", age, age)

    t := reflect.TypeOf(age)
    v := reflect.ValueOf(age)

    // 从接口变量到反射对象
    fmt.Printf("从接口变量到反射对象：Type对象的类型为 %T \n", t)
    fmt.Printf("从接口变量到反射对象：Value对象的类型为 %T \n", v)

}
```
输出如下
```Go
原始接口变量的类型为 int，值为 25 
从接口变量到反射对象：Type对象的类型为 *reflect.rtype
从接口变量到反射对象：Value对象的类型为 reflect.Value
如此我们完成了从接口类型变量到反射对象的转换。
```

等等，上面我们定义的 age 不是 int 类型的吗？第一法则里怎么会说是接口类型的呢？

关于这点，其实在上一节（ 35. 关于接口的三个"潜规则" ）已经提到过了，**由于 TypeOf 和 ValueOf 两个函数接收的是 interface{} 空接口类型，而 Go 语言函数都是值传递，因此Go语言会将我们的类型隐式地转换成接口类型**。
```Go
// TypeOf returns the reflection Type of the value in the interface{}.TypeOf returns nil.
func TypeOf(i interface{}) Type

// ValueOf returns a new Value initialized to the concrete value stored in the interface i. ValueOf(nil) returns the zero Value.
func ValueOf(i interface{}) Value
```

### 第二定律
> Reflection goes from reflection object to interface value.

和第一定律刚好相反，第二定律描述的是，从反射对象到接口变量的转换。

![](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/14/16287347117980.jpg)

通过源码可知， reflect.Value 的结构体会接收一个 Interface 方法，返回了一个 interface{}  类型的变量（注意：只有 Value 才能逆向转换，而 Type 则不行，这也很容易理解，如果 Type 能逆向，那么逆向成什么呢？）
```Go
// Interface returns v's current value as an interface{}.
// It is equivalent to:
//    var i interface{} = (v's underlying value)
// It panics if the Value was obtained by accessing
// unexported struct fields.
func (v Value) Interface() (i interface{}) {
    return valueInterface(v, true)
}
```
这个函数就是我们用来实现将反射对象转换成接口变量的一个桥梁。

例子如下
```Go
package main

import (
"fmt"
"reflect"
)

func main() {
    var age interface{} = 25

    fmt.Printf("原始接口变量的类型为 %T，值为 %v \n", age, age)

    t := reflect.TypeOf(age)
    v := reflect.ValueOf(age)

    // 从接口变量到反射对象
    fmt.Printf("从接口变量到反射对象：Type对象的类型为 %T \n", t)
    fmt.Printf("从接口变量到反射对象：Value对象的类型为 %T \n", v)

    // 从反射对象到接口变量
    i := v.Interface()
    fmt.Printf("从反射对象到接口变量：新对象的类型为 %T 值为 %v \n", i, i)
}
```
输出如下
```Go
原始接口变量的类型为 int，值为 25 
从接口变量到反射对象：Type对象的类型为 *reflect.rtype
从接口变量到反射对象：Value对象的类型为 reflect.Value
从反射对象到接口变量：新对象的类型为 int 值为 25 
```
当然了，最后转换后的对象，静态类型为 interface{} ，如果要转成最初的原始类型，需要再类型断言转换一下，关于这点，我已经在上一节里讲解过了，你可以点此前往复习：（ [35. 关于接口的三个"潜规则"](http://mp.weixin.qq.com/s?__biz=MzU1NzU1MTM2NA==&mid=2247483853&idx=1&sn=5b857a19b2b36f7f6096ce11a5f1df30&chksm=fc355ba6cb42d2b01c0d3112edc2662d81430ee59866024c5687a3d9fc040f6feb2da876e50e&scene=21#wechat_redirect)）。

`i := v.Interface().(int)`
至此，我们已经学习了反射的两大定律，对这两个定律的理解，我画了一张图，你可以用下面这张图来加强理解，方便记忆。

![](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/14/16287349471401.jpg)

### 第三定律
> To modify a reflection object, the value must be settable.

反射世界是真实世界的一个『映射』，是我的一个描述，但这并不严格，因为并不是你在反射世界里所做的事情都会还原到真实世界里。

第三定律引出了一个 `settable` （**可设置性，或可写性**）的概念。

其实早在以前的文章中，我们就一直在说，Go 语言里的函数都是值传递，只要你传递的不是变量的指针，你在函数内部对变量的修改是不会影响到原始的变量的。

回到反射上来，当你使用 reflect.Typeof 和 reflect.Valueof 的时候，如果传递的不是接口变量的指针，反射世界里的变量值始终将只是真实世界里的一个拷贝，你对该反射对象进行修改，并不能反映到真实世界里。

因此在反射的规则里

- 不是接收变量指针创建的反射对象，是不具备『可写性』的
- 是否具备『可写性』，可使用 CanSet() 来获取得知
- 对不具备『可写性』的对象进行修改，是没有意义的，也认为是不合法的，因此会报错。

```Go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var name string = "Go编程时光"
    v := reflect.ValueOf(name)
    fmt.Println("可写性为:", v.CanSet())
}
```

输出如下
```Go
可写性为: false
```
要让反射对象具备可写性，需要注意两点:
1. 创建反射对象时传入变量的指针
2. 使用 Elem()函数返回指针指向的数据

完整代码如下
```Go
package main

import (
    "fmt"
    "reflect"
)
func main() {
    var name string = "Go编程时光"
    v1 := reflect.ValueOf(&name)
    fmt.Println("v1 可写性为:", v1.CanSet())
    v2 := v1.Elem()
    fmt.Println("v2 可写性为:", v2.CanSet())
}
```

输出如下
```Go
v1 可写性为: false
v2 可写性为: true
```
知道了如何使反射的世界里的对象具有可写性后，接下来是时候了解一下如何对修改更新它。

反射对象，都会有如下几个以 Set 单词开头的方法

![](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/14/16287351794712.jpg)

这些方法就是我们修改值的入口。

来举个例子
```Go
package main

import (
    "fmt"
    "reflect"
)
func main() {
    var name string = "Go编程时光"
    fmt.Println("真实世界里 name 的原始值为：", name)
    v1 := reflect.ValueOf(&name)
    v2 := v1.Elem()
    v2.SetString("Python编程时光")
    fmt.Println("通过反射对象进行更新后，真实世界里 name 变为：", name)
}
```

输出如下
```Go
真实世界里 name 的原始值为：Go编程时光
通过反射对象进行更新后，真实世界里 name 变为：Python编程时光
```

以上便是我对反射三大定律的理解，我个人认为只要正确理解了这三条，反射便不再抽象。希望本文能对你有用。