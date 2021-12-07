作为 Golang 开发人员，遇到的众多问题之一是试图将函数的参数设为可选。这是一个非常常见的用例，有一些对象应该使用一些基本的默认设置开箱即用，并且您可能偶尔想要提供一些更详细的配置。

在python 中，你可以给参数一个默认值，并在调用方法时省略它们。但是在 Golang 中，是无法这么做。

那么怎么解决这个问题呢？ 答案就是Options模式。Options模式在Golang中应用十分广泛，几乎每个框架中都有他的踪迹。

## 传统方式：

```Go
package main

import "fmt"

type Person struct {
    Name string
    Age int
    Gender int
    Height int

    Country string
    Address string
}


func NewPerson(name string, age, gender, height int, country, address string) *Person{
    return &Person{
        Name: name,
        Age:age,
        Gender:gender,
        Height:height,
        Country:country,
        Address:address,
    }
}

func main()  {
    person := NewPerson("dongxiaojian", 18, 1, 180, "china", "Beijing")
    fmt.Printf("%+v\n", person)
}
```

可以发现，在这种方式中，拓展起来是非常麻烦的，以及设置出默认值十分繁琐。

## options模式

```Go
package main

import "fmt"

type Person struct {
 Name string
 Age int
 Gender int
 Height int

 Country string
 City    string
}

type Options func(*Person)

func WithPersonProperty (name string, age,gender,height int) Options {
 return func(p *Person){
     p.Name = name
     p.Age = age
     p.Gender = gender
     p.Height = height
 }
}

func WithRegional(country, city string) Options {
 return func(p *Person){
     p.Country = country
     p.City = city
 }
}

func NewPerson(opt ...Options) *Person{
 p := new(Person)
 p.Country = "china"
 p.City = "beijing"
 for _, o := range opt{
     o(p)
 }
 return p
}

func main()  {

 // 默认值方式
 person := NewPerson(WithPersonProperty("dongxiaojian", 18, 1, 180))
 fmt.Printf("%+v\n", person)

 // 设置值
 person2 := NewPerson(WithPersonProperty("dongxiaojian", 18, 1, 180),WithRegional("china", "hebei"))
 fmt.Printf("%+v\n", person2)
}
```

1. 下次可以通过`with**`函数进行增加属性，以及使用默认值。 这样看起来条理清晰了很多。