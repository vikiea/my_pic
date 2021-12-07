> map 是go 中一种很重要的 映射查找的数据结构，通过 key 的hash 运算来找到 值，这在各个语言中都不少见，这篇我们主要讲go map 的使用和其内部实现。


## map的使用

关于 map的使用问题， 如下

map的声明为:

/* 声明变量，默认 map 是 nil */
var map_variable map[key_data_type]value_data_type

/* 使用 make 函数 */
map_variable := make(map[key_data_type]value_data_type)
没有初始化的map不能进行插入操作，需要用make 函数进行初始化后才能进行操作。

map 的简单函数操作，curd为
```Go
var emptyMap map[string]int
var initMap = make(map[string]int)
//emptyMap["ok"]=1
initMap["init"] = 2
initMap["ok"] = 4
delete(initMap, "init")
for k, v := range initMap {
fmt.Printf("%v  ,%v\n", k, v)
}
_,texist := initMap["fine"]
fmt.Printf("exists find key is %v\n",texist)
fmt.Printf("%v ,%v,not init map is nil %v \n", emptyMap, initMap, emptyMap == nil)
```
输出情况为:
```
[1 ok [2 3 4]]
ok  ,4
exists find key is false
map[] ,map[ok:4],not init map is nil true 
```

map的操作都很简单，在这里不讲了。

值得注意的是，这里的**map 是非线程安全的，如果需要线程安全的map ，需要使用 sync.Map 或者自己用 sync.RWMute 自己来做锁保护。**

### map 的实现结构

> 通过搜索 hmap 的结构定义，我们在 reflect 中找到 hmap的定义，这个也就是 go 中map的实际内存结构定义, 在 runtime\map.go 中能找到相关定义

```go
// A header for a Go map.

const ( // 关键的变量
    bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits  // 一个桶最多存储8个key-value对
	loadFactorNum = 13 // 扩散因子：loadFactorNum / loadFactorDen = 6.5。
	loadFactorDen = 2  // 即元素数量 >= (hash桶数量(2^hmp.B) * 6.5 / 8) 时，触发扩容
)


type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8  // 记录几个特殊的位标记，如当前是否有别的线程正在写map、当前是否为相同大小的增长（扩容/缩容？）
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // 指向2^B个桶组成的数组的指针，数据存在这里
	oldbuckets unsafe.Pointer // 指向扩容前的旧buckets数组，只在map增长时有效
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
	
	extra *mapextra // 保存溢出桶的链表和未使用的溢出桶数组的首地址
}
// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap   
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap  // 为溢出的桶保留链表
}

// A bucket for a Go map.
type bmap struct {
	// tophash存储桶内每个key的hash值的高字节
	// tophash[0] < minTopHash表示桶的疏散状态
	// 当前版本bucketCnt的值是8，一个桶最多存储8个key-value对
	tophash [bucketCnt]uint8
	// 特别注意：
	// 实际分配内存时会申请一个更大的内存空间A，A的前8字节为bmap
	// 后面依次跟8个key、8个value、1个溢出指针
	// map的桶结构实际指的是内存空间A
}
```


可以看到 map的定义相比 slice 实在是复杂不少，我们盯着其中一点来写，map的实现结构无非就是桶机制，我们看看它的桶是怎么实现的，给一张图,


具体点 bmap 容器的内存结构应该是这样:

![20201112142136527](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/20/20201112142136527.png)



![20201112142208990](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/20/20201112142208990.png)

我们简单看下， 通过 key 获取值的流程,

附上源码 

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	hash := t.hasher(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if t.key.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}
```

核心在那个for 循环取值那，简单而言就是:

- **取得 hash 运算的 值 x, 通过  x 的低8位找到对应的bucket 桶号**

- **然后获得 x的高8位 ，依次和 bucket 里面的8个 高位key 做对比， 如果没找到匹配，然后和这个bucket 的溢出桶做对比，同样的流程。**

- **如果找到高8位相同，通过 指针运算直接找到 bucket那个key内存位置的值， 对比 bucket 中的key 和 输入的key值，如果相等那就算找到，如果不等，还得重复这个流程继续找。**

  **emptyResult 表示 高8位没有赋值，也就是这个 bucket之后 里面的某些key 和 value还没有塞入值。**

### map 的初始化

 使用`make(map[k]v, hint)`创建map时会调用`makemap()`函数，代码逻辑比较简单。
值得注意的是，`makemap()`创建的hash数组，数组的前面是hash表的空间，当`hint >= 4`时后面会追加`2^(hint-4)`个桶，之后再内存页帧对齐又追加了若干个桶（参见2.2章节结构图的hash数组部分）
所以**创建map时一次内存分配既分配了用户预期大小的hash数组，又追加了一定量的预留的溢出桶，还做了内存对齐**，一举多得。

```go
// make(map[k]v, hint), hint即预分配大小
// 不传hint时，如用new创建个预设容量为0的map时，makemap只初始化hmap结构，不分配hash数组
func makemap(t *maptype, hint int, h *hmap) *hmap {
	// 省略部分代码
	// 随机hash种子
	h.hash0 = fastrand()

	// 2^h.B 为大于hint*6.5(扩容因子)的最小的2的幂
	B := uint8(0)
	// overLoadFactor(hint, B)只有一行代码：return hint > bucketCnt && uintptr(hint) > loadFactorNum*(bucketShift(B)/loadFactorDen)
	// 即B的大小应满足 hint <= (2^B) * 6.5
	// 一个桶能存8对key-value，所以这就表示B的初始值是保证这个map不需要扩容即可存下hint个元素对的最小的B值
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B
	
	// 这里分配hash数组
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		// makeBucketArray()会在hash数组后面预分配一些溢出桶，
		// h.extra.nextOverflow用来保存上述溢出桶的首地址
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}
	
	return h
}

// 分配hash数组
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b) // base代表用户预期的桶的数量，即hash数组的真实大小
	nbuckets := base // nbuckets表示实际分配的桶的数量，>= base，这就可能会追加一些溢出桶作为溢出的预留
	if b >= 4 {
		// 这里追加一定数量的桶，并做内存对齐
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	// 后面的代码就是申请内存空间了，此处省略
	// 这里大家可以思考下这个数组空间要怎么分配，其实就是n*sizeof(桶)，所以：
		// 每个桶前面是8字节的tophash数组，然后是8个key，再是8个value，最后放一个溢出指针
		// sizeof(桶) = 8 + 8*sizeof(key) + 8*sizeof(value) + 8
	
	return buckets, nextOverflow
}
```





### 写入值和扩容

插入

1. 参数合法性检测，计算hash值

```go
unc mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // 不熟悉指针操作的同学，用指针传参往往会踩空指针的坑
    // 这里大家可以思考下，为什么h要非空判断？
    // 如果一定要在这里支持空map并检测到map为空时自动初始化，应该怎么写？
    // 提示：指针的指针
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	// 在这里做并发判断，检测到并发写时，抛异常
	// 注意：go map的并发检测是伪检测，并不保证所有的并发都会被检测出来。而且这玩意是在运行期检测。
	// 所以对map有并发要求时，应使用sync.map来代替普通map，通过加锁来阻断并发冲突
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	hash := alg.hash(key, uintptr(h.hash0)) // 这里得到uint32的hash值
	h.flags ^= hashWriting // 置Writing标志，key写入buckets后才会清除标志
	if h.buckets == nil { // map不能为空，但hash数组可以初始是空的，这里会初始化
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}
	...
}
```

2. 定位key在hash表中的位置

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    ...
again:
	bucket := hash & bucketMask(h.B) // 这里用hash值的低阶位定位hash数组的下标偏移量
	if h.growing() {
		growWork(t, h, bucket) // 这里是map的扩容缩容操作，我们在第4章单独讲
	}
	// 通过下标bucket，偏移定位到具体的桶
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	top := tophash(hash) // 这里取高8位用于在桶内定位键值对
	
	...
}
```

3. 进一步定位key可以插入的桶及桶中的位置

两轮循环，外层循环遍历hash桶及其指向的溢出链表，内层循环则在桶内遍历（一个桶最多8个key-value对）
有可能正好链表上的桶都满了，这时inserti为nil，第4步会链接一个新的溢出桶进来

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    ...

    var inserti *uint8          // tophash插入位置
    var insertk unsafe.Pointer  // key插入位置
    var val unsafe.Pointer      // value插入位置
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
				    // 找到个空位，先记录下tophash、key、value的插入位置
				    // 但要遍历完才能确定要不要插入到这个位置，因为后面有可能有重复的元素
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop // 遍历完整个溢出链表，退出循环
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if !alg.equal(key, k) {
				continue
			}
			// 走到这里说明map里找到一个重复的key，更新key-value，跳到第5步
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			goto done // 更新Key后跳到第5步
		}
		ovf := b.overflow(t)
		if ovf == nil {
			break // 遍历完整个溢出链表，没找到能插入的空位，结束循环，下一步再追加一个溢出桶进来
		}
		b = ovf // 继续遍历下一个溢出桶
	}

	...
}
```

4.  插入key


```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    ...
    // 扩容相关
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

	if inserti == nil { // inserti == nil说明上1步没找到空位，整个链表是满的，这里添加一个新的溢出桶上去
		newb := h.newoverflow(t, b) // 分配新溢出桶，优先用3.1章节预留的溢出桶，用完了则分配一个新桶内存
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		val = add(insertk, bucketCnt*uintptr(t.keysize))
	}
	
	// 当key或value的类型大小超过一定值时，桶只存储key或value的指针。这里分配空间并取指针
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectvalue() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(val) = vmem
	}
	typedmemmove(t.key, insertk, key) // 在桶中对应位置插入key
	*inserti = top // 插入tophash，hash值高8位
	h.count++ // 插入了新的键值对，h.count数量+1
	
	...
}
5. 结束插入

func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    ...
    
 done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting // 释放hashWriting标志位
	if t.indirectvalue() {
		val = *((*unsafe.Pointer)(val))
	}
	return val // 返回value可插入位置的指针，注意，value还没插入
}
mapassign()只插入tophash和key，并返回val指针，编译器会在调用mapassign()后用汇编往val插入value
```

### 扩容

关于扩容相关代码在 mapassign 那里

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    ...
    
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
    	hashGrow(t, h)
    	goto again // Growing the table invalidates everything, so try again
    }
    
    ...
}

// overLoadFactor()返回true则触发扩容，即map的count大于hash桶数量(2^B)*6.5
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}

// tooManyOverflowBuckets()，顾名思义，溢出桶太多了触发缩容
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	if B > 15 {
		B = 15
	}
	return noverflow >= uint16(1)<<(B&15)
}
```

解释：

> map只在插入元素即mapassign()函数中对是否扩容缩容进行触发，条件即是上面这段代码：

- 条件1：当前不处在growing状态
- 条件2-1：触发扩容：map的数据量count大于hash桶数量(2B)*6.5。注意这里的(2B)只是hash数组大小，不包括溢出的桶
- 条件2-2：触发缩容：溢出的桶数量noverflow>=32768(1<<15)或者>=hash数组大小。


#### map 缩容是伪缩容

现在可以解释为什么我把map的缩容叫做伪缩容了：**因为缩容仅仅针对溢出桶太多的情况，触发缩容时hash数组的大小不变，即hash数组所占用的空间只增不减。**

也就是说，如果我们把一个已经增长到很大的map的元素挨个全部删除掉，hash表所占用的内存空间也不会被释放。

所以**如果要实现“真缩容”，需自己实现缩容搬迁，即创建一个较小的map，将需要缩容的map的元素挨个搬迁过来**：

```go
// go map缩容代码示例
myMap := make(map[int]int, 1000000)

// 假设这里我们对bigMap做了很多次插入，之后又做了很多次删除，此时bigMap的元素数量远小于hash表大小
// 接下来我们开始缩容
smallMap := make(map[int]int, len(myMap))
for k, v := range myMap {
    smallMap[k] = v
}
myMap = smallMap // 缩容完成，原来的map被我们丢弃，交给gc去清理
```

#### 删除元素操作

代码为

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
    ...

            b.tophash[i] = emptyOne // 先标记删除
    		// 如果b.tophash[i]不是最后一个元素，则暂时先占着坑。emptyOne标记的位置暂时不能被插入新元素(见3.2章节插入函数)
    		if i == bucketCnt-1 {
    			if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
    				goto notLast
    			}
    		} else {
    			if b.tophash[i+1] != emptyRest {
    				goto notLast
    			}
    		}
    		for { // 如果b.tophash[i]是最后一个元素，则把末尾的emptyOne全部清除置为emptyRest
    			b.tophash[i] = emptyRest
    			if i == 0 {
    				if b == bOrig {
    					break // beginning of initial bucket, we're done.
    				}
    				// Find previous bucket, continue at its last entry.
    				c := b
    				for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
    				}
    				i = bucketCnt - 1
    			} else {
    				i--
    			}
    			if b.tophash[i] != emptyOne {
    				break
    			}
    		}
    
    ...
}
```
 删除与插入类似，前面的步骤都是参数和状态判断、定位key-value位置，然后clear对应的内存。不展开说。以下是几个关键点：

- 删除过程中也会置hashWriting标志
- 当key/value过大时，hash表里存储的是指针，这时候用软删除，置指针为nil，数据交给gc去删。当然，这是map的内部处理，外层是无感知的，拿到的都是值拷贝
- 无论Key/value是值类型还是指针类型，删除操作都只影响hash表，外层已经拿到的数据不受影响。尤其是指针类型，外层的指针还能继续使用


由于定位key位置的方式是查找tophash，所以删除操作对tophash的处理是关键：

- map首先将对应位置的tophash[i]置为emptyOne，表示该位置已被删除
- 如果tophash[i]不是整个链表的最后一个，则只置emptyOne标志，该位置被删除但未释放，后续插入操作不能使用此位置
- 如果tophash[i]是链表最后一个有效节点了，则把链表最后面的所有标志为emptyOne的位置，都置为emptyRest。置为emptyRest的位置可以在后续的插入操作中被使用。
- 这种删除方式，以少量空间来避免桶链表和桶内的数据移动。事实上，go 数据一旦被插入到桶的确切位置，map是不会再移动该数据在桶中的位置了。


总结
> Go 语言使用拉链法来解决哈希碰撞的问题实现了哈希表，它的访问、写入和删除等操作都在编译期间转换成了运行时的函数或者方法。

- map 底层是hash实现，数据结构为**hash数组 + 桶 + 溢出的桶链表**，每个桶存储最多8个key-value对

- 查找和插入的原理：key的hash值（低阶位）与桶数量相与，得到key所在的hash桶，再用key的高8位与桶中的tophash[i]对比，相同则进一步对比key值，key值相等则找到

- go map不支持并发。插入、删除、搬迁等操作会置writing标志，检测到并发直接panic

- 每次扩容hash表增大1倍，hash表只增不减

- 支持有限缩容，delete操作只置删除标志位，释放溢出桶的空间依靠触发缩容来实现。

- map在使用前必须初始化，否则panic：已初始化的map是make(map[key]value)或make(map[key]value, hint)这两种形式。而new或var xxx map[key]value这两种形式是未初始化的，直接使用会panic。

 

go采用的hash算法应是很成熟的算法，极端情况暂不考虑。所以综合情况下go map的时间复杂度应为O(1)

空间复杂度分析：
首先我们不考虑因删除大量元素导致的空间浪费情况（这种情况现在go是留给程序员自己解决），只考虑一个持续增长状态的map的一个空间使用率：
由于溢出桶数量超过hash桶数量时会触发缩容，所以最坏的情况是数据被集中在一条链上，hash表基本是空的，这时空间浪费O(n)。
最好的情况下，数据均匀散列在hash表上，没有元素溢出，这时最好的空间复杂度就是扩散因子决定了，当前go的扩散因子由全局变量决定，即loadFactorNum/loadFactorDen = 6.5。即平均每个hash桶被分配到6.5个元素以上时，开始扩容。所以最小的空间浪费是(8-6.5)/8 = 0.1875，即O(0.1875n)

> 结论：go map的空间复杂度（指除去正常存储元素所需空间之外的空间浪费）是O(0.1875n) ~ O(n)之间。
