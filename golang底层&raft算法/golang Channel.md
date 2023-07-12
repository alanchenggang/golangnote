## 场景

>一道考题
>要求实现一个 map:
>   (1) 面向高并发;
>   (2) 只存在插入和查询操作 0(1);
>   (3)查询时，若 key 存在，直接返回 val;
>           若 key 不存在，阻塞直到 key val 对被放入后，获取 val 返回;
>           等待指定时长仍未放入，返回超时错误;
>   (4) 写出真实代码，不能有死锁或者 panic 风险

```go

```

#### Q1.Channel的核心数据结构

```go
//创建channel
ch:=make(chan int) //无缓冲类型
ch:=make(chan int，10) // 有缓冲 缓冲区大小为10

// channel 读
val := <- ch
<-ch //直接读丢弃
val,ok := <- ch

// channel 写
var data type
ch<-data 

//关闭
close(ch) 

// 1. 读已关闭的channel
读不会阻塞，如果该channel中仍有数据，则会把channel中的数据读出来；如果该channel中没有数据，则会读到当前channel数据类型的零值
// 2. 写已关闭的channel
向该channel中写数据会发生panic

// select多路复用结合channel进行读写操作 
// 同时监听多分枝，哪个分枝有事件，则解除阻塞继续执行
select{
    case <-parent.Done():
		// do something
    case <-child.Done():
    	// do something
    case grandpa <-data:
    	// do something
    default:
    // 所有的case都没能达成条件 执行放行，让代码继续运行
}
```

```go
type hchan struct {
	qcount   uint           // 当前 channel中存在多少个元素
	dataqsiz uint           // 当前 channel 能存放的元素容量
	buf      unsafe.Pointer // channel中用于存放先素的环形缓冲区
	elemsize uint16 // channel 元素类型的大小
	closed   uint32 // 标识 channel 是否关闭 
    				//	channel 重复关闭 -----> panic
	elemtype *_type //  channel元素类型
	sendx    uint   // 发送元素进入环形缓冲区的 index
	recvx    uint   // 接收元素所处的环形缓冲区的 index
	recvq    waitq  // 因接收而陷入阻塞的协程队列
	sendq    waitq  // 因发送而陷入阻塞的协程队列;


	//这个锁保护了 hchan 中的所有字段，以及在此 channel 上被阻塞的一些 sudog 的字段
	// 警告在持有锁的时候不要改变另一个 Goroutine 的状态，特别是不要 ready 一个 Goroutine，因为这可能会导致堆栈缩小时死锁
	lock mutex
}

type waitq struct {
	first *sudog // 队列头部
	last  *sudog // 队列尾部
}


// sudog 表示在等待列表中的 Goroutine，例如在 channel 上发送/接收时
//
// sudog 的存在是因为 Goroutine 和同步对象之间的关系是多对多的。
//一个 Goroutine 可以在多个等待列表中，因此可能有多个 sudog 对应一个 Goroutine
// 许多 Goroutine 可能在等待同一个同步对象，因此可能有多个 sudog 对应一个对象
//
// sudog 是从一个特殊的池中分配的，使用 acquireSudog 和 releaseSudog 来分配和释放它们
type sudog struct {
// sudog 中的一些字段是由该 sudog 阻塞的 channel 的 hchan.lock 保护的
//shrinkstack 依赖于这个锁来处理与 channel 操作相关的 sudog
//这意味着在使用这些字段时，需要先获取该 channel 的锁，以确保对这些字段的访问是安全的

	g *g

	next *sudog
	prev *sudog
	elem unsafe.Pointer // data element (may point to stack)

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool

	// success indicates whether communication over channel c
	// succeeded. It is true if the goroutine was awoken because a
	// value was delivered over channel c, and false if awoken
	// because c was closed.
	success bool

	parent   *sudog // semaRoot binary tree
	waitlink *sudog // g.waiting list or semaRoot
	waittail *sudog // semaRoot
	c        *hchan // channel
}

```

![56f449721f85aef3e756517348aee915](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/56f449721f85aef3e756517348aee915.png)

#### Q2.Channel的构造流程

```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

    // 计算总空间大小 避免超出容量
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
        // 1. 元素大小为0 -----> struct{} 信号传达
        // 2. 元素个数为0 -----》 无缓冲
 	case mem == 0:
		// Queue or element size is zero.
            //hchanSize 固定值 96Byte大小
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
        //mallocgc 申请内存空间函数
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size) // 总容量大小
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}
```

####  Q3.Channel的写流程
