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

##### Q3. 引用传递



##### Q4. 截取操作



##### Q5. append操作



##### Q6. 扩容流程



##### Q7. 删除操作/拷贝操作



##### Q8. 总结

