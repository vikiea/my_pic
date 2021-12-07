# Go开发相关疑问点

#### 如何打印SQL语句到日志
```Go
if provinceName, ok := entity.ShowProvince[item.Province]; ok {
   item.ProvinceName = provinceName
}
```
#### 如何判断变量是否存在

#### 如何修改range里数组的值
因为Go是值拷贝,且range里的变量是临时变量,所以需要通过修改原始变量赋值
```Go
for key, item := range *zyFreightList {
	if provinceName, ok := entity.ShowProvince[item.Province]; ok {
		(*zyFreightList)[key].ProvinceName = provinceName
	}
}
```

#### 如何在golang中解析相对路径到绝对路径?
##### 获取当前目录
```Go
os.GetPWD()
filepath.Abs(path) # 绝对目录
filepath.Dir(path) # 相对目录
可以 filepath.Abs(filepath.Dir(path))
```
##### 获取字符目录,前缀,后缀等方法
```Go
filepath.Split(path)
filepath.Base(path) # test.txt
filepath.Ext(path) # .txt
```
For example:

```Go
fmt.Println(filepath.Abs("/home/bob/../alice"))
```
Outputs:

`/home/alice <nil>`

```Go
a := filepath.Base("storage/cache/js/zy_freight_sel.js")
b := filepath.Dir("storage/cache/js/zy_freight_sel.js")
c, _ := filepath.Abs("storage/cache/js/zy_freight_sel.js")
d, _ := os.Getwd()
e, _ := filepath.Split("storage/cache/js/zy_freight_sel.js")
f := filepath.Ext("storage/cache/js/zy_freight_sel.js")
dir, _ := filepath.Abs(filepath.Dir(os.Args[0]))
_, g, _, _ := runtime.Caller(0)
fmt.Println(a, b, c, d, e, f, dir, g)
```
![-w1293](https://gitee.com/vikieq/my_pic/raw/master/uPic/LHxTX1Jyve7AkMm.jpg)

##### Cannot assign to map错误
如代码:
```Go
package main

import "fmt"

func main() {
	type Entity struct {
		Value string
	}

	entityMap := make(map[string]Entity, 0)
	entityMap["cat"] = Entity{Value: "This is a cat"}
	entityMap["cat"].Value= "This is a another cat"
	fmt.Println("value ",entityMap["cat"].Value)
}
```
第12行编译报错 ： `cannot assign to struct field entityMap[“cat”].Value in map`
原因: **map 元素是无法取址的**，也就说可以得到 a[“tao”], 但是无法对其进行修改
解决办法:**使用指针类型的map**

```Go
package main

import "fmt"

func main() {
   fmt.Println("Hello, World!")
	type Entity struct {
		Value string
	}

	entityMap := make(map[string]*Entity, 2)
	entityMap["cat"] = &Entity{Value: "This is a cat"}
	
	
	entityMap["cat"].Value = "This is a another cat"
	fmt.Println("value ",entityMap["cat"].Value)
}
```
> **golang里面的map是通过hashtable来实现的**，具体方式就是通过拉链法（数组+链表）.对比c++的map,c++里面的map， 是通过红黑树来实现的。所以二者在遍历的时候做删除操作，golang的是可以直接操作的，因为内部实现是哈希映射，删除并不影响其他项，而c++中的map删除，由于是红黑树，删除任意一项，都会打乱迭代指针，不能再O(1)时间内删除。
> 
> 同时，**golang里面的key是无序的**，即使你顺序添加，遍历的时候也是无序。
> 
> **golang里面的map,当通过key获取到value时，这个value是不可寻址的**，因为map 会进行**动态扩容**，当进行扩展后，map的value就会进行**内存迁移**，其地址发生变化，所以无法对这个value进行寻址。也就是造成上述问题的原因所在。
> 
> map的扩容与slice不同，那么map本身是引用类型，作为形参或返回参数的时候，传递的是值的拷贝，而值是地址，**扩容时也不会改变这个地址**。**而slice的扩容，会导致地址的变化。**
> map的本质是散列表，而map的增长扩容会导致重新进行散列，这就可能使map的遍历结果在扩容前后变得不可靠，Go设计者为了让大家不依赖遍历的顺序，故意在实现map遍历时加入了随机数，让每次遍历的起点--即起始bucket的位置不一样，即不让遍历都从bucket0开始，所以即使未扩容时我们遍历出来的map也总是无序的。

##### 以读写模式打开同一文件句柄,写入内容后无法立即读出来

##### 控制器全局构造函数怎么实现
在路由时直接初始化所有属性

##### 验证器validate中,绑定参数验证时的tag起到什么作用
- json控制输出格式,同时控制shouldBindJSON时候的输入格式,该方法对应content-type/json请求
- form对应表单提交方式,可通过shouldBind自动判断

##### 插入数据库时time.Time类型传什么值
time.Now()即可

##### 插入数据库时,create,save方法如何传值
必须传入指针才可以,否则报不可寻址错误

##### 堆和栈是什么区别
- 堆用来存放用户(开发者)创建的变量,一般是全局变量
- Go里面局部变量存储在栈(stack)上(通常说的堆栈就是栈,后进先出,与数据结构中的栈类似)
- 全局变量存储在堆(heap)上,其数据结构类似于链表,非数据结构中的堆

##### proto是什么

##### Golang中协程与线程等关系
golang的runtime调度是依赖pmg的角色抽象，p为逻辑处理器（执行上下文），m为执行体(OS线程)，g为协程。p的runq队列中放着可执行的goroutine结构。golang默认p的数量为cpu core数目，比如物理核心为8cpu core，那么go processor的数量就为8。另外，同一时间一个p只能绑定一个m线程，pm绑定后自然就找g和运行g。可以通过`runtime.GOMAXPROCS(runtime.NumCPU())`更改最大并行进程数量

##### FirstOrCreate使用有问题
```Go
data := entity.MySignboard{
		Uid:    uid,
		Status: 1,
		Step:   2,
	}
data2 := &entity.MySignboard{}
err := m.db.Select("uid,status,step").Where(data).FirstOrCreate(&data2).Error
```
输出:
```SQL
SELECT uid,status,step FROM `my_signboard` WHERE `my_signboard`.`uid` = 500047392 AND `my_signboard`.`status` = 1 AND `my_signboard`.`step` = 2 ORDER BY `my_signboard`.`id` LIMIT 1;
INSERT INTO `my_signboard` VALUES()
```

##### 验证通过model更新完数据库后model是否会同步更新
通过指针传值,会
##### panic和捕获异常,为什么捕获不到?
recover()用于将panic的信息捕捉。**recover必须定义在panic之前的defer语句中**
```Go
defer func() {
	if r := recover(); r != nil {
		log.Fatal("啊不好意思,程序崩溃啦,哈哈哈哈哈")
	}
}()
```