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



##### Q6. 扩容流程



##### Q7. 删除操作/拷贝操作



##### Q8. 总结

