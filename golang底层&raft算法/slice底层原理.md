### slice技术原理与应用实战

##### Q1. 数据结构

```go
type slice struct {
    // 指向起点的地址
    array unsafe.Pointer
    // 切片长度
    len   int
    // 切片容量
    cap   int
}
```

![image-20230806145751899](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230806145751899.png)

>- array：指向了内存空间地址的起点. 由于 slice 数据存放在连续的内存空间中，后续可以根据索引 index，在起点的基础上快速进行地址偏移，从而定位到目标元素
>
>- len：切片的长度，指的是逻辑意义上 slice 中实际存放了多少个元素
>
>- cap：切片的容量，指的是物理意义上为 slice 分配了足够用于存放多少个元素的空间. 使用 slice 时，要求 cap 永远大于等于 len
>
>  通过 slice 数据结构定义可以看到，每个 slice header 中存放的是内存空间的地址（array 字段），后续在传递切片的时候，相当于是对 slice header 进行了一次值拷贝，但内部存放的地址是相同的，因此对于 slice 本身属于引用传递操作(浅拷贝)

##### Q2. 初始化

- 声明但不初始化

  ```go
    var s []int 
  ```

  此时 s==nil

- 基于make初始化

  - 初始化时 len == cap

    ```go
      s := make([]int,8)
    ```

    此时会将切片的长度 len 和 容量 cap 同时设置为 8. 切片的长度一旦被指定了，就代表对应位置已经被分配了元素，尽管设置的会是对应元素类型下的零值.

  - 初始化时 len < cap

    ```go
      s := make([]int,8,16)
    ```

    代表已经在切片中设置了 8 个元素，会设置为对应类型的零值；cap = 16 代表为 slice 分配了用于存放 16 个元素的空间. 需要保证 cap >= len. 在 index 为 [len, cap) 的范围内，虽然内存空间已经分配了，但是逻辑意义上不存在元素，直接访问会 panic 报数组访问越界；但是访问 [0,len) 范围内的元素是能够正常访问到的，只不过会是对应元素类型下的零值.

- 初始化连带赋值

  ```go
    s := []int{2,3,4}
  ```

  代表 slice 长度 len 和容量 cap 均设置为 3，同时完成对这 3 个元素赋值.

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
    // 根据 cap 结合每个元素的大小，计算出消耗的总容量
    mem, overflow := math.MulUintptr(et.size, uintptr(cap))
    if overflow || mem > maxAlloc || len < 0 || len > cap {
        // 倘若容量超限，len 取负值或者 len 超过 cap，直接 panic
        mem, overflow := math.MulUintptr(et.size, uintptr(len))
        if overflow || mem > maxAlloc || len < 0 {
            panicmakeslicelen()
        }
        panicmakeslicecap()
    }
    // 走 mallocgc 进行内存分配以及切片初始化
    return mallocgc(mem, et, true)
}
```

##### Q3. 引用传递(类似于深拷贝浅拷贝??)

![image-20230806150734435](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230806150734435.png)

引用传递，指的是，将实例的地址信息传递到方法中，这样在方法中会直接通过地址追溯到实例所在位置，因此执行的一些修改操作会直接影响到原实例.

![image-20230806150809920](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230806150809920.png)

值传递，指的是对实例进行一轮拷贝，得到一个副本，然后将这个副本传递到方法中. 这样在方法内部发生的修改动作都作用于这个副本之上，而副本本身和实例是相互独立的，因此不会影响到原实例.

测试用例:

```go
func Test_slice(t *testing.T){
  s := []int{2,3,4}
  // [2,3,4] -> [-1,3,4]
  changeSlice(s)
}
func changeSlice(s []int){
  s[0] = -1
}

----> 切片的传递是引用传递，而非值传递
```

每个切片实例对应一个 slice header，其中存储了三个字段：

- 切片内存空间的起始地址 array；

- 切片长度 len；

- 切片容量 cap.

每次我们在方法间传递切片时，会对 slice header 实例本身进行一次值拷贝，然后将 slice header 的副本传递到局部方法中.然而，这个 slice header 副本中的 array 和原 slice 指向同一片内存空间，因此在局部方法中执行修改操作时，还会根据这个地址信息影响到原 slice 所属的内存空间，从而对内容发生影响.

##### Q4. 截取操作

可以修改 slice 下标的方式，进行 slice 内容的截取，形如 s[a:b] 的格式，其中 a b 代表切片的索引 index，左闭右开，比如 s[a:b] 对应的范围是 [a,b)，代表的是取切片 slice index = a ~ index = b-1 范围的内容.

此外，a 和 b 是可以缺省的：

- 如果 a 缺省不填则默认取 0 ，则代表从切片起始位置开始截取. 比如 s[:b] 等价于 s[0:b]
- 如果 b 缺省不填，则默认取 len(s)，则代表末尾截取到切片长度 len 的终点，比如 s[a:] 等价于 s[a:len(s)]
- a 和 b 均缺省也是可以的，则代表截取整个切片长度的范围，比如 s[:] 等价于 s[0:len(s)]

```go
func Test_slice(t *testing.T){
   s := []int{1,2,3,4,5}
   // s1: [2,3,4,5]
   s1 := s[1:]
   // s2: [1,2,3,4]
   s2 := s[:len(s)-1]
   // s3: [2,3,4] 
   s3 := s[1:len(s)-1]
   // ...
}
```

在对切片 slice 执行截取操作时，本质上是一次引用传递操作，因为不论如何截取，底层复用的都是同一块内存空间中的数据，只不过，截取动作会创建出一个新的 slice header 实例.

```go
  s := []int{2,3,4,5}
  s1 := s[1:]
  // ...
```



![image-20230806151337386](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230806151337386.png)

s1 = s[1:] 的操作，会创建出一个 s1 的 slice header，其中的字段 array 会在 s.array 的基础上向右偏移一个切片元素大小的数值；s1.len 和 s1.cap 也会以 s1 的起点为起点，以 s 原定的 len 和 cap 终点为终点，最终推算得出 s1.len = s.len - 1；s1.cap = s.cap - 1.

##### Q5. append操作

> 通过 append 操作，可以在 slice 末尾，额外新增一个元素. 需要注意，这里的末尾指的是针对 slice 的长度 len 而言. 这个过程中倘若发现 slice 的剩余容量已经不足了，则会对 slice 进行扩容. 

```go
func Test_slice(t *testing.T){
    s := []int{2,3,4}  
    s = append(s,5)
    // s: [2,3,4,5]
}

```

在创建 slice 时，如果能够预估到其未来所需的容量空间，则应该提前分配好对应容量，避免在运行过程中频繁触发扩容操作，这样会对性能产生不利的影响.

错误代码示例:

```go
func Test_slice(t *testing.T){
    s := make([]int,5)
    for i := 0; i < 5; i++{
       s = append(s, i)
    }
    // 结果为：
    // s: [0,0,0,0,0,0,1,2,3,4]
}

预期的操作时声明出一个长度为 5 的 slice，同时依次向其中填入 0,1,2,3,4 的五个元素，然而按照上述代码执行下来，得到的结果是事与愿违的，其原因在于
 1. 通过 make 操作，声明了一个长度和容量均为 5 的切片 s，此时前 5 个元素已经被填充为零值
 2. 接下来执行 append 操作时，只会在长度末尾进行追加. 最终会引发扩容，并最终得到结果为 [0,0,0,0,0,0,1,2,3,4,5]
```

两个正确实例:

1. 设置不同的长度 len 和容量 cap 值

   ```go
   func Test_slice(t *testing.T){
       s := make([]int,0,5)
       for i := 0; i < 5; i++{
          s = append(s, i)
       }
       // 结果为：
       // s: [0,1,2,3,4]
   }
   ```

2. 通过遍历 slice 的方式进行执行位置元素的赋值

   ```go
   func Test_slice(t *testing.T){
       s := make([]int,0,5)
       for i := 0; i < 5; i++{
          s = append(s, i)
       }
       // 结果为：
       // s: [0,1,2,3,4]
   }
   ```

##### Q6. 扩容流程

>- 倘若扩容后预期的新容量小于原切片的容量，则 panic
>- 倘若切片元素大小为 0（元素类型为 struct{}），则直接复用一个全局的 zerobase 实例，直接返回
>- 倘若预期的新容量超过老容量的两倍，则直接采用预期的新容量
>- 倘若老容量小于 256，则直接采用老容量的2倍作为新容量
>- 倘若老容量已经大于等于 256，则在老容量的基础上扩容 1/4 的比例并且累加上 192 的数值，持续这样处理，直到得到的新容量已经大于等于预期的新容量为止
>- 结合 mallocgc 流程中，对内存分配单元 mspan 的等级制度，推算得到实际需要申请的内存空间大小
>- 调用 mallocgc，对新切片进行内存初始化
>- 调用 memmove 方法，将老切片中的内容拷贝到新切片中
>- 返回扩容后的新切片

```go
func growslice(et *_type, old slice, cap int) slice {
    //... 
    if cap < old.cap {
        panic(errorString("growslice: cap out of range"))
    }


    if et.size == 0 {
        // 倘若元素大小为 0，则无需分配空间直接返回
        return slice{unsafe.Pointer(&zerobase), old.len, cap}
    }


    // 计算扩容后数组的容量
    newcap := old.cap
    // 取原容量两倍的容量数值
    doublecap := newcap + newcap
    // 倘若新的容量大于原容量的两倍，直接取新容量作为数组扩容后的容量
    if cap > doublecap {
        newcap = cap
    } else {
        const threshold = 256
        // 倘若原容量小于 256，则扩容后新容量为原容量的两倍
        if old.cap < threshold {
            newcap = doublecap
        } else {
            // 在原容量的基础上，对原容量 * 5/4 并且加上 192
            // 循环执行上述操作，直到扩容后的容量已经大于等于预期的新容量为止
            for 0 < newcap && newcap < cap {             
                newcap += (newcap + 3*threshold) / 4
            }
            // 倘若数值越界了，则取预期的新容量 cap 封顶
            if newcap <= 0 {
                newcap = cap
            }
        }
    }


    var overflow bool
    var lenmem, newlenmem, capmem uintptr
    // 基于容量，确定新数组容器所需要的内存空间大小 capmem
    switch {
    // 倘若数组元素的大小为 1，则新容量大小为 1 * newcap.
    // 同时会针对 span class 进行取整
    case et.size == 1:
        lenmem = uintptr(old.len)
        newlenmem = uintptr(cap)
        capmem = roundupsize(uintptr(newcap))
        overflow = uintptr(newcap) > maxAlloc
        newcap = int(capmem)
    // 倘若数组元素为指针类型，则根据指针占用空间结合元素个数计算空间大小
    // 并会针对 span class 进行取整
    case et.size == goarch.PtrSize:
        lenmem = uintptr(old.len) * goarch.PtrSize
        newlenmem = uintptr(cap) * goarch.PtrSize
        capmem = roundupsize(uintptr(newcap) * goarch.PtrSize)
        overflow = uintptr(newcap) > maxAlloc/goarch.PtrSize
        newcap = int(capmem / goarch.PtrSize)
    // 倘若元素大小为 2 的指数，则直接通过位运算进行空间大小的计算   
    case isPowerOfTwo(et.size):
        var shift uintptr
        if goarch.PtrSize == 8 {
            // Mask shift for better code generation.
            shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
        } else {
            shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
        }
        lenmem = uintptr(old.len) << shift
        newlenmem = uintptr(cap) << shift
        capmem = roundupsize(uintptr(newcap) << shift)
        overflow = uintptr(newcap) > (maxAlloc >> shift)
        newcap = int(capmem >> shift)
    // 兜底分支：根据元素大小乘以元素个数
    // 再针对 span class 进行取整     
    default:
        lenmem = uintptr(old.len) * et.size
        newlenmem = uintptr(cap) * et.size
        capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
        capmem = roundupsize(capmem)
        newcap = int(capmem / et.size)
    }




    // 进行实际的切片初始化操作
    var p unsafe.Pointer
    // 非指针类型
    if et.ptrdata == 0 {
        p = mallocgc(capmem, nil, false)
        // ...
    } else {
        // 指针类型
        p = mallocgc(capmem, et, true)
        // ...
    }
    // 将切片的内容拷贝到扩容后的位置 p 
    memmove(p, old.array, lenmem)
    return slice{p, old.len, newcap}
}
```



##### Q7. 删除操作/拷贝操作

从切片中删除元素的实现思路，本质上和切片内容截取的思路是一致的.

如果需要删除 slice 中间的某个元素，操作思路则是采用内容截取加上元素追加的复合操作，可以先截取待删除元素的左侧部分内容，然后在此基础上追加上待删除元素后侧部分的内容：

```go
func Test_slice(t *testing.T){
    s := []int{0,1,2,3,4}
    // 删除 index = 2 的元素
    s = append(s[:2],s[3:]...)
    // s: [0,1,3,4], len: 4, cap: 5
    t.Logf("s: %v, len: %d, cap: %d", s, len(s), cap(s))
}
```

最后，当我们需要删除 slice 中的所有元素时，也可以采用切片内容截取的操作方式：s[:0]. 这样操作后，slice header 中的指针 array 仍指向远处，但是逻辑意义上其长度 len 已经等于 0，而容量 cap 则仍保留为原值.

slice 的拷贝可以分为简单拷贝和完整拷贝两种类型.

要实现简单拷贝，我们只需要对切片的字面量进行赋值传递即可，这样相当于创建出了一个新的 slice header 实例，但是其中的指针 array、容量 cap 和长度 len 仍和老的 slice header 实例相同.

```go
func Test_slice3(t *testing.T) {
	s := []int{0, 1, 2, 3, 4}
	s1 := s
	t.Logf("address of s: %p, address of s1: %p", s, s1)
}
```

![image-20230807223248025](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230807223248025.png)

**切片的截取操作也属于是简单拷贝**

slice 的完整复制，指的是会创建出一个和 slice 容量大小相等的独立的内存区域，并将原 slice 中的元素一一拷贝到新空间中.

在实现上，slice 的完整复制可以调用系统方法 copy，代码示例如下，通过日志打印的方式可以看到，s 和 s1 的地址是相互独立的：

```go
func Test_slice(t *testing.T) {
    s := []int{0, 1, 2, 3, 4}
    s1 := make([]int, len(s))
    copy(s1, s)
    t.Logf("s: %v, s1: %v", s, s1)
    t.Logf("address of s: %p, address of s1: %p", s, s1)
}
```

![image-20230807223401456](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230807223401456.png)

##### Q8. 总结

- slice 是一个长度可变的连续数据序列，在实现上基于一个 slice header 组成，其中包含的字段包括：指向内存空间地址起点的指针 array、一个表示了存储数据长度的 len 和分配空间长度的 cap

- 由于 slice 在传递过程中，本质上传递的是 slice header 实例中的内存地址 array，因此属于引用传递

- slice 在扩容时，遵循如下机制:

  • 如果扩容时预期的新容量超过原容量的两倍，直接取预期的新容量

  • 如果原容量小于 256，直接取原容量的两倍作为新容量

  • 如果原容量大于等于 256，在原容量 n 的基础上循环执行 n += (n+3*256)/4 的操作，直到 n 大于等于预期新容量，并取 n 作为新容量

-  **slice 不是并发安全的数据结构**
