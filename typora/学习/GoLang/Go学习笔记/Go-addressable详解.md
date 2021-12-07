# go addressable 详解

Go语言规范中规定了可寻址(addressable)对象的定义,

> For an operand x of type T, the address operation &x generates a pointer of type *T to x. The operand must be addressable, that is, either a variable, pointer indirection, or slice indexing operation; or a field selector of an addressable struct operand; or an array indexing operation of an addressable array. As an exception to the addressability requirement, x may also be a (possibly parenthesized) composite literal. If the evaluation of x would cause a run-time panic, then the evaluation of &x does too.

对于一个对象`x`, 如果它的类型为`T`, 那么`&x`则会产生一个类型为`*T`的指针，这个指针指向`x`, 这是这一段的第一句话，也是我们在开发过程中经常使用的一种获取对象指针的一种方式。



[![img](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/14/memory.jpg)](https://colobu.com/2018/02/27/go-addressable/memory.jpg)

## addressable

上面规范中的这段话规定， `x`必须是可寻址的， 也就是说，它只能是以下几种方式：

- 一个变量: `&x`
- 指针引用(pointer indirection): `&*x`
- slice索引操作(不管slice是否可寻址): `&s[1]`
- 可寻址struct的字段: `&point.X`
- 可寻址数组的索引操作: `&a[0]`
- composite literal类型: `&struct{ X int }{1}`

下列情况`x`是不可以寻址的，你不能使用`&x`取得指针：

- 字符串中的字节:
- map对象中的元素
- 接口对象的动态值(通过type assertions获得)
- 常数
- literal值(非composite literal)
- package 级别的函数
- 方法method (用作函数值)
- 中间值(intermediate value):
  - 函数调用
  - 显式类型转换
  - 各种类型的操作 （除了指针引用pointer dereference操作`*x`):
    - channel receive operations
    - sub-string operations
    - sub-slice operations
    - 加减乘除等运算符

> Tapir Games在他的文章[unaddressable-values](http://www.tapirgames.com/blog/golang-unofficial-faq#unaddressable-values)中做了很好的整理。

有几个点需要解释下：

- 常数为什么不可以寻址?： 如果可以寻址的话，我们可以通过指针修改常数的值，破坏了常数的定义。
- map的元素为什么不可以寻址？:两个原因，如果对象不存在，则返回零值，零值是不可变对象，所以不能寻址，如果对象存在，因为Go中map实现中元素的地址是变化的，这意味着寻址的结果是无意义的。
- 为什么slice不管是否可寻址，它的元素读是可以寻址的？:因为slice底层实现了一个数组，它是可以寻址的。
- 为什么字符串中的字符/字节又不能寻址呢：因为字符串是不可变的。

规范中还有几处提到了 `addressable`:

- 调用一个receiver为指针类型的方法时，使用一个addressable的值将自动获取这个值的指针
- `++`、`--`语句的操作对象必须是addressable或者是map的index操作
- 赋值语句`=`的左边对象必须是addressable,或者是map的index操作，或者是`_`
- 上条同样使用`for ... range`语句

## `reflect.Value`的`CanAddr`方法和`CanSet`方法

在我们使用`reflect`执行一些底层的操作的时候， 比如编写序列化库、rpc框架开发、编解码、插件开发等业务的时候，经常会使用到`reflect.Value`的`CanSet`方法，用来动态的给对象赋值。 `CanSet`比`CanAddr`只加了一个限制，就是struct类型的`unexported`的字段不能`Set`，所以我们这节主要介绍`CanAddr`。

并不是任意的`reflect.Value`的`CanAddr`方法都返回`true`,根据它的godoc,我们可以知道：

> CanAddr reports whether the value's address can be obtained with Addr. Such values are called addressable. A value is addressable if it is an element of a slice, an element of an addressable array, a field of an addressable struct, or the result of dereferencing a pointer. If CanAddr returns false, calling Addr will panic.

也就是只有下面的类型`reflect.Value`的`CanAddr`才是`true`, 这样的值是`addressable`:

- slice的元素
- 可寻址数组的元素
- 可寻址struct的字段
- 指针引用的结果

与规范中规定的`addressable`, `reflect.Value`的`addressable`范围有所缩小， 比如对于栈上分配的变量， 随着方法的生命周期的结束， 栈上的对象也就被回收掉了，这个时候如果获取它们的地址，就会出现不一致的结果，甚至安全问题。

> 对于栈和堆的对象分配以及逃逸分析，你可以看 William Kennedy 写的系列文章: [Go 语言机制之逃逸分析](https://studygolang.com/articles/12444)

所以如果你想通过`reflect.Value`对它的值进行更新，应该确保它的`CanSet`方法返回`true`,这样才能调用`SetXXX`进行设置。

使用`reflect.Value`的时候有时会对`func Indirect(v Value) Value`和`func (v Value) Elem() Value`两个方法有些迷惑，有时候他们俩会返回同样的值，有时候又不会。

总结一下：

1. 如果`reflect.Value`是一个指针， 那么`v.Elem()`等价于`reflect.Indirect(v)`
2. 如果不是指针
   2.1 如果是`interface`, 那么`reflect.Indirect(v)`返回同样的值，而`v.Elem()`返回接口的动态的值
   2.2 如果是其它值, `v.Elem()`会panic,而`reflect.Indirect(v)`返回原值

下面的代码列出一些`reflect.Value`是否可以addressable, 你需要注意数组和struct字段的情况，也就是`x7`、`x9`、`x14`、`x15`的正确的处理方式。

```go
package main

import (
	"fmt"
	"reflect"
	"time"
)

func main() {
	checkCanAddr()
}

type S struct {
	X int
	Y string
	z int
}

func M() int {
	return 100
}

var x0 = 0

func checkCanAddr() {
	// 可寻址的情况
	v := reflect.ValueOf(x0)
	fmt.Printf("x0: %v \tcan be addressable and set: %t, %t\n", x0, v.CanAddr(), v.CanSet()) //false,false

	var x1 = 1
	v = reflect.Indirect(reflect.ValueOf(x1))
	fmt.Printf("x1: %v \tcan be addressable and set: %t, %t\n", x1, v.CanAddr(), v.CanSet()) //false,false

	var x2 = &x1
	v = reflect.Indirect(reflect.ValueOf(x2))
	fmt.Printf("x2: %v \tcan be addressable and set: %t, %t\n", x2, v.CanAddr(), v.CanSet()) //true,true

	var x3 = time.Now()
	v = reflect.Indirect(reflect.ValueOf(x3))
	fmt.Printf("x3: %v \tcan be addressable and set: %t, %t\n", x3, v.CanAddr(), v.CanSet()) //false,false

	var x4 = &x3
	v = reflect.Indirect(reflect.ValueOf(x4))
	fmt.Printf("x4: %v \tcan be addressable and set: %t, %t\n", x4, v.CanAddr(), v.CanSet()) // true,true

	var x5 = []int{1, 2, 3}
	v = reflect.ValueOf(x5)
	fmt.Printf("x5: %v \tcan be addressable and set: %t, %t\n", x5, v.CanAddr(), v.CanSet()) // false,false

	var x6 = []int{1, 2, 3}
	v = reflect.ValueOf(x6[0])
	fmt.Printf("x6: %v \tcan be addressable and set: %t, %t\n", x6[0], v.CanAddr(), v.CanSet()) //false,false

	var x7 = []int{1, 2, 3}
	v = reflect.ValueOf(x7).Index(0)
	fmt.Printf("x7: %v \tcan be addressable and set: %t, %t\n", x7[0], v.CanAddr(), v.CanSet()) //true,true

	v = reflect.ValueOf(&x7[1])
	fmt.Printf("x7.1: %v \tcan be addressable and set: %t, %t\n", x7[1], v.CanAddr(), v.CanSet()) //true,true

	var x8 = [3]int{1, 2, 3}
	v = reflect.ValueOf(x8[0])
	fmt.Printf("x8: %v \tcan be addressable and set: %t, %t\n", x8[0], v.CanAddr(), v.CanSet()) //false,false

	// https://groups.google.com/forum/#!topic/golang-nuts/RF9zsX82MWw
	var x9 = [3]int{1, 2, 3}
	v = reflect.Indirect(reflect.ValueOf(x9).Index(0))
	fmt.Printf("x9: %v \tcan be addressable and set: %t, %t\n", x9[0], v.CanAddr(), v.CanSet()) //false,false

	var x10 = [3]int{1, 2, 3}
	v = reflect.Indirect(reflect.ValueOf(&x10)).Index(0)
	fmt.Printf("x9: %v \tcan be addressable and set: %t, %t\n", x10[0], v.CanAddr(), v.CanSet()) //true,true

	var x11 = S{}
	v = reflect.ValueOf(x11)
	fmt.Printf("x11: %v \tcan be addressable and set: %t, %t\n", x11, v.CanAddr(), v.CanSet()) //false,false

	var x12 = S{}
	v = reflect.Indirect(reflect.ValueOf(&x12))
	fmt.Printf("x12: %v \tcan be addressable and set: %t, %t\n", x12, v.CanAddr(), v.CanSet()) //true,true

	var x13 = S{}
	v = reflect.ValueOf(x13).FieldByName("X")
	fmt.Printf("x13: %v \tcan be addressable and set: %t, %t\n", x13, v.CanAddr(), v.CanSet()) //false,false

	var x14 = S{}
	v = reflect.Indirect(reflect.ValueOf(&x14)).FieldByName("X")
	fmt.Printf("x14: %v \tcan be addressable and set: %t, %t\n", x14, v.CanAddr(), v.CanSet()) //true,true

	var x15 = S{}
	v = reflect.Indirect(reflect.ValueOf(&x15)).FieldByName("z")
	fmt.Printf("x15: %v \tcan be addressable and set: %t, %t\n", x15, v.CanAddr(), v.CanSet()) //true,false

	v = reflect.Indirect(reflect.ValueOf(&S{}))
	fmt.Printf("x15.1: %v \tcan be addressable and set: %t, %t\n", &S{}, v.CanAddr(), v.CanSet()) //true,true

	var x16 = M
	v = reflect.ValueOf(x16)
	fmt.Printf("x16: %p \tcan be addressable and set: %t, %t\n", x16, v.CanAddr(), v.CanSet()) //false,false

	var x17 = M
	v = reflect.Indirect(reflect.ValueOf(&x17))
	fmt.Printf("x17: %p \tcan be addressable and set: %t, %t\n", x17, v.CanAddr(), v.CanSet()) //true,true

	var x18 interface{} = &x11
	v = reflect.ValueOf(x18)
	fmt.Printf("x18: %v \tcan be addressable and set: %t, %t\n", x18, v.CanAddr(), v.CanSet()) //false,false

	var x19 interface{} = &x11
	v = reflect.ValueOf(x19).Elem()
	fmt.Printf("x19: %v \tcan be addressable and set: %t, %t\n", x19, v.CanAddr(), v.CanSet()) //true,true

	var x20 = [...]int{1, 2, 3}
	v = reflect.ValueOf([...]int{1, 2, 3})
	fmt.Printf("x20: %v \tcan be addressable and set: %t, %t\n", x20, v.CanAddr(), v.CanSet()) //false,false
}
```

## 参考资料

- http://www.tapirgames.com/blog/golang-unofficial-faq#unaddressable-values
- https://blog.golang.org/laws-of-reflection
- https://stackoverflow.com/questions/25384640/why-golang-reflect-makeslice-returns-un-addressable-value
- http://grokbase.com/t/gg/golang-nuts/13bwpzedxh/go-nuts-golang-reflect-addressable-is-not-consistency
- https://github.com/golang/go/issues/11865
- https://stackoverflow.com/questions/20224478/dereferencing-a-map-index-in-golang
- https://stackoverflow.com/questions/38168329/why-are-map-values-not-addressable
- https://stackoverflow.com/questions/40793289/go-struct-literals-why-is-this-one-addressable
- https://groups.google.com/forum/#!topic/golang-nuts/FnTCX9R_PbM
- https://stackoverflow.com/questions/24318389/golang-elem-vs-indirect-in-the-reflect-package