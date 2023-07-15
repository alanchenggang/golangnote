![img](https://oscimg.oschina.net/oscnet/up-44f9427f2b61f085b9326ae3cf514a73e2c.JPEG)

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
type ImproveChannel struct {
	sync.Once //确保只关闭一次
	ch        chan struct{}
}

func NewImproveChannel() *ImproveChannel {
	return &ImproveChannel{
		ch: make(chan struct{}),
	}
}

func (channel *ImproveChannel) Close() {
	channel.Once.Do(func() {
		close(channel.ch)
	})
}

type ConcurrentMap struct {
	m          map[string]string
	improveCh  map[string]*ImproveChannel
	sync.Mutex //互斥锁
}

func NewConcurrentMap() *ConcurrentMap {
	return &ConcurrentMap{
		m:         make(map[string]string),
		improveCh: make(map[string]*ImproveChannel),
	}
}

func (concurrentMap *ConcurrentMap) Put(key string, val string) {
	concurrentMap.Lock()
	defer concurrentMap.Unlock()
	concurrentMap.m[key] = val
	channel, ok := concurrentMap.improveCh[key]
	if !ok {
		return
	}
	channel.Close()
}

func (concurrentMap *ConcurrentMap) Get(key string, maxWaitDuration time.Duration) (string, error) {
	concurrentMap.Lock()
	val, ok := concurrentMap.m[key]
	if ok {
		concurrentMap.Unlock()
		return val, nil
	}
	channel, ok := concurrentMap.improveCh[key]
	// 不存在则创建新的channel
	if !ok {
		channel = NewImproveChannel()
		concurrentMap.improveCh[key] = channel
	}
	ctx, cannel := context.WithTimeout(context.Background(), maxWaitDuration)
	defer cannel()
	concurrentMap.Unlock()
	select {
	case <-ctx.Done():
		return "", ctx.Err()
	case <-channel.ch:
	}
	concurrentMap.Lock()
	val = concurrentMap.m[key]
	concurrentMap.Unlock()
	return val, nil
}

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

##### channel写两类异常情况处理

```go
func chansend1(c *hchan, elem unsafe.Pointer) {
    chansend(c, elem, true, getcallerpc())
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    //对于未初始化的 chan，写入操作会引发死锁；
    if c == nil {
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }

    lock(&c.lock)
	//对于已关闭的 chan，写入操作会引发 panic.
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
    
    // ...
```

##### CASE1. 写时存在阻塞读协程

```GO
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // ...
	//加锁
    lock(&c.lock)

    // ...
	//从阻塞度协程队列中取出一个 goroutine 的封装对象 sudog
    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        //在 send 方法中，会基于 memmove 方法，直接将元素拷贝交给 sudog 对应的 goroutine 在 send 方法中会完成解锁动作.
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }
    
    // ...
```

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/20230716010623.png)

##### CASE2. 写时无阻塞读协程但环形缓冲区仍有空间

```GO
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 加锁
    lock(&c.lock)
    // ...
    if c.qcount < c.dataqsiz {
        // 将当前元素添加到环形缓冲区 sendx 对应的位置.
        qp := chanbuf(c, c.sendx)
        typedmemmove(c.elemtype, qp, ep)
        // 更新channel参数
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }

    // ...
}
```

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/20230716010644.png)

##### CASE3. 写时无阻塞读协程且环形缓冲区无空间(阻塞当前写协程)

```GO
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 加锁
    lock(&c.lock)

    // 构造封装当前 goroutine 的 sudog 对象
    gp := getg()
    mysg := acquireSudog()
    mysg.elem = ep
    mysg.g = gp
    mysg.c = c
    //完成指针指向，建立 sudog、goroutine、channel 之间的指向关系
    gp.waiting = mysg
    //把 sudog 添加到当前 channel 的阻塞写协程队列中
    c.sendq.enqueue(mysg)
    //park 当前协程
    atomic.Store8(&gp.parkingOnChan, 1)
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
    //倘若协程从 park 中被唤醒，则回收 sudog（sudog能被唤醒，其对应的元素必然已经被读协程取走）
    gp.waiting = nil
    closed := !mysg.success
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    //解锁，返回
    return true
}
```

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/20230716010715.png)

##### 写流程整体

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/20230716002830.png)

#### Q4. 读流程

##### CASE1. 读空 channel

```GO
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    if c == nil {
        // park 挂起，引起死锁
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    // ...
}
```

##### CASE2. channel 已关闭且内部无元素

```GO
unc chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
  
    lock(&c.lock)
//直接解锁返回
    if c.closed != 0 {
        if c.qcount == 0 {
            unlock(&c.lock)
            if ep != nil {
                typedmemclr(c.elemtype, ep)
            }
            return true, false
        }
        // The channel has been closed, but the channel's buffer have data.
    } 

    // ...
```

##### CASE3. 读时有阻塞的写协程

```GO
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
   
    lock(&c.lock)

    //从阻塞写协程队列中获取到一个写协程
    if sg := c.sendq.dequeue(); sg != nil {
        //倘若 channel 无缓冲区，则直接读取写协程元素，并唤醒写协程
        // 倘若 channel 有缓冲区，则读取缓冲区头部元素，并将写协程元素写入缓冲区尾部后唤醒写协程
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
     }
     // ...
}
```

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/20230716004151.png)

##### CASE4. 读时无阻塞写协程且缓冲区有元素

```GO
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // ...
    lock(&c.lock)
    // ...
    if c.qcount > 0 {
        // 获取到 recvx 对应位置的元素
        qp := chanbuf(c, c.recvx)
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp)
        // 更新channel属性
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }
    // ...
```

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/20230716004327.png)

##### CASE5. 读时无阻塞写协程且缓冲区无元素

```GO
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
   // ...
   lock(&c.lock)
   // ...
    gp := getg()
    //构造封装当前 goroutine 的 sudog 对象
    mysg := acquireSudog()
    mysg.elem = ep
    //完成指针指向，建立 sudog、goroutine、channel 之间的指向关系
    gp.waiting = mysg
    mysg.g = gp
    mysg.c = c
    gp.param = nil
    //把 sudog 添加到当前 channel 的阻塞读协程队列中
    c.recvq.enqueue(mysg)
    atomic.Store8(&gp.parkingOnChan, 1)
    //park 当前协程
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), 
    waitReasonChanReceive, traceEvGoBlockRecv, 2)
	//倘若协程从 park 中被唤醒，则回收 sudog（sudog能被唤醒，其对应的元素必然已经被写入）
    gp.waiting = nil
    success := mysg.success
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, success
}
```

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/20230716004656.png)

##### 读流程整体

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/20230716004725.png)

#### Q5. 非阻塞模式

>非阻塞模式下，读/写 channel 方法通过一个 bool 型的响应参数，用以标识是否读取/写入成功.
>
>• 所有需要使得当前 goroutine 被挂起的操作，在非阻塞模式下都会返回 false；
>
>• 所有是的当前 goroutine 会进入死锁的操作，在非阻塞模式下都会返回 false；
>
>• 所有能立即完成读取/写入操作的条件下，非阻塞模式下会返回 true.

默认情况下，读/写 channel 都是阻塞模式，只有在**select** 语句组成的多路复用分支中，与 channel 的交互会变成非阻塞模式：

```go
ch := make(chan int)
select{
  case <- ch:
  default:
}
```

#### Q6.  两种读 channel 的协议

读取 channel 时，可以根据第二个 bool 型的返回值用以判断当前 channel 是否已处于关闭状态（零值判断）：

```GO
ch := make(chan int, 2)
got1 := <- ch
got2,ok := <- ch
```

#### Q7.  关闭channel

```go
func closechan(c *hchan) {
    // 关闭未初始化过的 channel 会 panic
    if c == nil {
        panic(plainError("close of nil channel"))
    }

    lock(&c.lock)
    //重复关闭 channel 会 panic
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }

    c.closed = 1

    var glist gList
     //将阻塞读协程队列中的协程节点统一添加到 glist
    // release all readers
    for {
        sg := c.recvq.dequeue()
        if sg == nil {
            break
        }

        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        gp := sg.g
        gp.param = unsafe.Pointer(sg)
        sg.success = false
        glist.push(gp)
    }
	//将阻塞写协程队列中的协程节点统一添加到 glist
    // release all writers (they will panic)
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        gp := sg.g
        gp.param = unsafe.Pointer(sg)
        sg.success = false
        glist.push(gp)
    }
    unlock(&c.lock)

    // 唤醒 glist 当中的所有协程
    for !glist.empty() {
        gp := glist.pop()
        gp.schedlink = 0
        goready(gp, 3)
```

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/20230716005131.png)
