# Go语言中的类型断言
> Go里的类型断言只能针对接口对象使用,不能对拥有具体类型的对象使用

## 1. Type Assertion
Type Assertion（中文名叫：类型断言），通过它可以做到以下几件事情
- 检查 i 是否为 nil
- 检查 i 存储的值是否为某个类型

具体的使用方式有两种：

### 第一种：
`t := i.(T)`
这个表达式可以断言一个接口对象（i）里不是 nil，并且接口对象（i）存储的值的类型是 T，如果断言成功，就会返回值给 t，如果断言失败，就会触发 panic。

### 第二种
`t, ok:= i.(T)`

## 2. Type Switch
如果需要区分多种类型，可以使用 type switch 断言，这个将会比一个一个进行类型断言更简单、直接、高效。
```Go
package main

import "fmt"

func findType(i interface{}) {
    switch x := i.(type) {
    case int:
        fmt.Println(x, "is int")
    case string:
        fmt.Println(x, "is string")
    case nil:
        fmt.Println(x, "is nil")
    default:
        fmt.Println(x, "not type matched")
    }
}

func main() {
    findType(10)      // int
    findType("hello") // string

    var k interface{} // nil
    findType(k)

    findType(10.23) //float64
}
```
输出如下
```Go
10 is int
hello is string
<nil> is nil
10.23 not type matched
```
额外说明一下：
- 如果你的值是 nil，那么匹配的是 case nil
- 如果你的值在 switch-case 里并没有匹配对应的类型，那么走的是 default 分支





