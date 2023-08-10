#### Q1. 线程与协程

> 线程是操作系统中的概念，是操作系统调度的基本单位。
>
> 每个线程都有自己的堆栈和寄存器等状态信息，线程之间的切换需要操作系统的介入。
>
> 线程的创建和销毁都需要操作系统的系统调用，因此线程的创建和销毁比较耗时。
>
> 线程之间的通信需要使用操作系统提供的同步原语，如锁、信号量等。

> 协程是一种轻量级的线程，也称为用户态线程。(与线程存在映射关系 为M:1)
>
> 协程的调度是由用户程序自己控制的，不需要操作系统的介入。
>
> 协程之间的切换是由程序自己控制的，因此切换的代价比线程小得多。
>
> 协程的创建和销毁都是由程序自己控制的，因此创建和销毁的代价也比线程小得多。
>
> 协程之间的通信可以使用共享内存或消息传递等方式。

#### Q2. goroutine 简介

```go
goroutine` 是 Go 语言中的轻量级线程，它由 Go 运行时环境管理。`goroutine` 可以在单个线程上运行，因此创建和销毁 `goroutine` 的成本很低。Go 语言的并发模型是基于 `goroutine` 和通道（channel）的，这使得编写并发程序变得更加容易和直观。在 Go 语言中，可以使用 `go` 关键字来启动一个新的 `goroutine
```
#### Q3.Golang的GMP原理与调度

##### Q3.01 Golang调度器的由来

1.  单进程时代不需要调度器
2.  多进程 / 线程时代有了调度器需求
3.  协程来提高 CPU 利用率

##### Q3.02 线程和协程绑定的三种方式

1. N 个协程绑定 1 个线程
2. 1 个协程绑定 1 个线程
3. M 个协程绑定 1 个线程

##### Q3.03 Go 语言的协程 goroutine
Go 为了提供更容易使用的并发方法，使用了 goroutine 和 channel。goroutine 来自协程的概念，让一组可复用的函数运行在一组线程之上，即使有协程阻塞，该线程的其他协程也可以被 runtime 调度，转移到其他可运行的线程上。最关键的是，程序员看不到这些底层的细节，这就降低了编程的难度，提供了更容易的并发。

Go 中，协程被称为 goroutine，它非常轻量，一个 goroutine 只占几 KB，并且这几 KB 就足够 goroutine 运行完，这就能在有限的内存空间内支持大量 goroutine，支持了更多的并发。虽然一个 goroutine 的栈只占几 KB，但实际是可伸缩的，如果需要更多内容，runtime 会自动为 goroutine 分配。

Goroutine 特点：

- 占用内存更小（几 kb）
- 调度更灵活 (runtime 调度)

#### Q4. 三种模型对比

| 模型      | 弱依赖内核 | 可并行 | 可应对阻塞 | 栈可动态扩缩 |
| --------- | ---------- | ------ | ---------- | ------------ |
| 线程      | N          | Y      | Y          | N            |
| 协程      | Y          | N      | N          | N            |
| goroutine | Y          | Y      | Y          | Y            |

#### Q5. gmp 理论模型

g-->goroutine

> golang中对协程的抽象
>
> 每个都有自己的运行栈&状态&执行的任务函数
>
> g需要绑定到p才能执行，在g的视角看来 p=== cpu

m-->machine

>golang中线程的抽象
>
>m不直接执行g，而是先和p绑定，由其实现代理
>
>因为p的存在，m无需和g绑定，也无需记录g的状态信息，因此g在全生命周期中可以实现跨m执行

p -->processor

> golang中的调度器
>
> gmp的中枢，实现g到m之间的动态结合
>
> 对g而言 p是cpu/对m而言 p是其执行代理，为其提供必要信息的同时隐藏了复杂的调度细节
>
> p的数量决定了g最大并行数量

#### Q6. gmp 数据结构

```go
    // G status
	//
	// G 的状态不仅仅表示它的一般状态，还充当了锁的作用
    // 锁住了 goroutine 的堆栈（stack），从而限制了它执行用户代码的能力
	//在 Go 语言中，每个 goroutine 都有自己的堆栈，用于存储它的局部变量和函数调用信息等。当一个 goroutine 被创建时，它的堆栈会被分配一定的内存空间。当 goroutine 执行函数时，它的局部变量和函数调用信息等都会存储在堆栈中。当函数返回时，这些信息会从堆栈中弹出，堆栈的内存空间也会被释放。
	// 如果要向 G（goroutine）的状态列表中添加新的状态，也需要在 mgcmark.go 文件中的“垃圾回收期间允许的状态”列表中添加相应的状态
	//在 Go 语言中，垃圾回收（garbage collection）是自动进行的，它会在程序运行时自动回收不再使用的内存。在垃圾回收期间，Go 运行时会遍历整个内存堆，标记出所有仍然被引用的对象，并将未被引用的对象释放掉。为了避免在垃圾回收期间出现问题，Go 运行时需要保证所有的 goroutine 都处于一些特定的状态，这些状态被称为“垃圾回收期间允许的状态”。
	//在 Go 语言中，垃圾回收器需要扫描所有的 goroutine，以确定哪些 goroutine 正在使用堆栈。为了实现这个功能，Go 运行时会在 G 的状态中添加一个 _Gscan 标志，表示这个 goroutine 正在被扫描。在扫描过程中，如果发现一个 goroutine 的状态为 _Grunnable，那么它就会被加入到运行队列中，等待下一次调度
	//_Gscan 标志的实现可以更加轻量级。例如，对于运行队列中的 _Gscanrunnable 状态的 goroutine，我们可以选择不运行它们，而是等待它们变成 _Grunnable 状态后再运行。这样可以避免使用 CAS 循环等待，从而提高程序的性能。.
	// 一些状态转换，例如 _Gscanwaiting 到 _Gscanrunnable，实际上不会影响堆栈的所有权，因此可以被允许。这也是一种优化措施，可以减少不必要的状态转换，提高程序的性能

	// 在 Go 语言中，G 的状态 _Gidle 表示这个 goroutine 刚刚被分配，但还没有被初始化
	// 在 goroutine 被分配时，Go 运行时会为它分配一些内存空间，但这些内存空间并没有被初始化，也就是说，它们的值是未定义的。在 goroutine 被初始化之前，它的状态会被设置为 _Gidle。
	//这个状态只是 goroutine 生命周期中的一个短暂阶段，一旦 goroutine 被初始化，它的状态就会变为 _Grunnable，可以被调度执行
	_Gidle = iota // 0

	// 在 Go 语言中，G 的状态 _Grunnable 表示这个 goroutine 已经被加入到运行队列中，等待被调度执行。
	// 在 _Grunnable 状态下，goroutine 并没有在执行用户代码，它的堆栈也没有被锁住，因此可以被调度器随时调度执行。
	// 在 _Grunnable 状态下，goroutine 的堆栈是没有所有权的，也就是说，它可以被多个线程共享。这是因为在 Go 语言中，goroutine 的调度是由 Go 运行时环境管理的，它会自动将 goroutine 分配到不同的线程上执行，从而实现并发执行。
	// _Grunnable 状态表示一个已经准备好被调度执行的 goroutine，它的堆栈没有被锁住，可以被多个线程共享。这个状态是 goroutine 生命周期中的一个重要阶段，它标志着 goroutine 已经准备好执行用户代码，等待被调度器调度执行。
	_Grunnable // 1

	// 在 Go 语言中，G 的状态 _Grunning 表示这个 goroutine 正在执行用户代码。在 _Grunning 状态下，goroutine 拥有自己的堆栈，堆栈的所有权归这个 goroutine 所有。这个 goroutine 不在运行队列中，也就是说，它不需要等待被调度器调度执行，而是直接执行用户代码。
	// 在 _Grunning 状态下，这个 goroutine 被分配了一个 M（machine）和一个 P（processor），它们分别表示执行 goroutine 的线程和处理器。在 _Grunning 状态下，goroutine 的 g.m 和 g.m.p 字段是有效的，可以用于获取当前执行 goroutine 的线程和处理器信息。
	// _Grunning 状态表示一个正在执行用户代码的 goroutine，它拥有自己的堆栈，堆栈的所有权归这个 goroutine 所有。这个状态是 goroutine 生命周期中的一个重要阶段，它标志着 goroutine 正在执行用户代码，处于活跃状态。
	_Grunning // 2

	// 在 Go 语言中，G 的状态 _Gsyscall 表示这个 goroutine 正在执行系统调用，而不是执行用户代码。在 _Gsyscall 状态下，goroutine 拥有自己的堆栈，堆栈的所有权归这个 goroutine 所有。这个 goroutine 不在运行队列中，也就是说，它不需要等待被调度器调度执行，而是直接执行系统调用。
	// 在 _Gsyscall 状态下，这个 goroutine 被分配了一个 M（machine），它表示执行 goroutine 的线程。在 _Gsyscall 状态下，goroutine 的 g.m 字段是有效的，可以用于获取当前执行 goroutine 的线程信息。
	// _Gsyscall 状态表示一个正在执行系统调用的 goroutine，它拥有自己的堆栈，堆栈的所有权归这个 goroutine 所有。这个状态是 goroutine 生命周期中的一个重要阶段，它标志着 goroutine 正在执行系统调用，而不是执行用户代码。
	_Gsyscall // 3

	// 在 Go 语言中，G 的状态 _Gwaiting 表示这个 goroutine 被阻塞在运行时中，而不是执行用户代码。在 _Gwaiting 状态下，goroutine 不在运行队列中，也不会被调度器调度执行。相反，它被记录在某个地方（例如，一个通道等待队列）中，以便在必要时可以被唤醒。
	// 在 _Gwaiting 状态下，goroutine 的堆栈不是被这个 goroutine 所拥有的，除非在通道操作期间，读取或写入堆栈的某些部分需要在适当的通道锁下进行。否则，在 goroutine 进入 _Gwaiting 状态后，访问堆栈是不安全的（例如，它可能会被移动）。
	// _Gwaiting 状态表示一个被阻塞在运行时中的 goroutine，它不在运行队列中，也不会被调度器调度执行。这个状态是 goroutine 生命周期中的一个重要阶段，它标志着 goroutine 正在等待某些事件的发生，例如等待通道的读取或写入。
	_Gwaiting // 4

	// 在 Go 语言中，G 的状态 _Gmoribund_unused 目前没有被使用，但是在 gdb 脚本中硬编码了这个状态。这个状态可能是为了以后的扩展而保留的，或者是为了兼容性而保留的。
	_Gmoribund_unused // 5

	//在 Go 语言中，G 的状态 _Gdead 表示这个 goroutine 目前没有被使用。它可能是刚刚退出，或者在空闲列表中，或者正在初始化。在 _Gdead 状态下，goroutine 不在运行队列中，也不会被调度器调度执行。它不执行用户代码，可能有堆栈分配，也可能没有。
	// 在 _Gdead 状态下，这个 goroutine 和它的堆栈（如果有的话）都归属于正在退出这个 goroutine 的 M，或者是从空闲列表中获取这个 goroutine 的 M 所拥有。
	//_Gdead 状态表示一个目前没有被使用的 goroutine，它可能是刚刚退出，或者在空闲列表中，或者正在初始化。这个状态是 goroutine 生命周期中的一个短暂阶段，它标志着 goroutine 目前没有被使用，等待被重新分配或释放。
	_Gdead // 6

type g struct {
	// 在 Go 语言中，G 的堆栈参数包括以下几个部分：
	// stack 描述了实际的堆栈内存，它是一个区间 [stack.lo, stack.hi) 
	// stackguard0 是在 Go 堆栈增长序言中与堆栈指针进行比较的值。它通常是 stack.lo+StackGuard，但是也可以是 StackPreempt，以触发抢占。
    // stackguard1 是在 C 堆栈增长序言中与堆栈指针进行比较的值。在 g0 和 gsignal 堆栈上，它是 stack.lo+StackGuard。在其他 goroutine 堆栈上，它是 ~0，以触发调用 morestackc（并崩溃）。
    //堆栈参数是 goroutine 生命周期中的一个重要部分，它描述了 goroutine 的堆栈内存区间以及堆栈指针的比较值
	stack       stack   // offset known to runtime/cgo
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink

	...
	m         *m      // 负责执行当前g的m; offset known to arm liblink
	sched     gobuf
    
    // 这三个字段都是在 goroutine 生命周期中的不同阶段使用的，用于保存和检查堆栈指针和程序计数器的值
	syscallsp uintptr // if status==Gsyscall, syscallsp = sched.sp to use during gc
	syscallpc uintptr // if status==Gsyscall, syscallpc = sched.pc to use during gc
	stktopsp  uintptr // expected sp at top of stack, to check in traceback

//`param` 字段是一个通用的指针参数字段，用于在特定上下文中传递值，如果在其他地方寻找参数的存储空间会很困难。目前，`param` 字段有以下三种用途：
//1. 当通道操作唤醒一个被阻塞的 `goroutine` 时，它将 `param` 设置为指向已完成阻塞操作的 `sudog`。
//2. 由 `gcAssistAlloc1` 使用，向其调用者发出信号，表示 `goroutine` 已完成 GC 循环。在其他方式下这样做是不安全的，因为 `goroutine` 的堆栈可能已经在此期间移动。
//3. 由 `debugCallWrap` 使用，将参数传递给新的 `goroutine`，因为在运行时分配闭包是被禁止的。
//，`param` 字段是一个通用的指针参数字段，用于在特定上下文中传递值。它在不同的上下文中有不同的用途，但都是为了解决特定的问题而设计的。
      
	param        unsafe.Pointer
	...
}


type gobuf struct {
	//在 gobuf 结构体中，sp、pc 和 g 字段的偏移量是已知的（在 libmach 中硬编码）。这些字段在 gobuf 结构体中用于保存堆栈指针、程序计数器和 goroutine 的指针。
//ctxt 字段与 GC 相关，因为它可能是一个堆分配的 funcval，所以 GC 需要跟踪它。但是，在汇编中设置和清除 ctxt 是很困难的，因为它需要写屏障。但是，ctxt 实际上是一个保存的、活动的寄存器，我们只在真正的寄存器和 gobuf 之间交换它。因此，在堆栈扫描期间，我们将其视为根，这意味着保存和恢复它的汇编代码不需要写屏障。但是，它仍然被定义为指针类型，以便从 Go 中的任何其他写入都会有写屏障。
	sp   uintptr
	pc   uintptr
	g    guintptr
	ctxt unsafe.Pointer
	ret  uintptr
	lr   uintptr
	bp   uintptr // for framepointer-enabled architectures
}

type m struct {
    //在 Go 语言中，m 表示一个内核线程（kernel thread），它负责调度和执行 goroutine。每个 m 都有一个 g0 字段，它是一个特殊的 goroutine，用于执行一些特殊的任务，例如垃圾回收和栈增长。
	//具体来说，g0 是一个全局的、长期存在的 goroutine，它不会被销毁或重新分配。它的堆栈大小是固定的，并且在启动时就已经分配好了。g0 的主要作用是在 m 启动时执行一些初始化任务，例如设置信号处理程序、初始化调度器等。此外，g0 还用于执行垃圾回收和栈增长等特殊任务。
	//在 m 中，g0 字段是一个指向 g（goroutine）结构体的指针，它表示当前正在执行的 goroutine。由于 g0 是一个全局的、长期存在的 goroutine，因此它的状态和生命周期与其他 goroutine 不同。
	g0      *g     // goroutine with scheduling stack
    ...
    //在 Go 语言中，m 表示一个内核线程（kernel thread），它负责调度和执行 goroutine。每个 m 都有一个 tls 字段，它是一个指向线程本地存储（Thread Local Storage，TLS）的指针。
	// TLS 是一种线程级别的存储机制，它允许每个线程都有自己的私有数据。在 Go 语言中，m 的 tls 字段用于存储一些与线程相关的数据，例如当前的 g（goroutine）、当前的 P（处理器）、当前的 defer 栈等。这些数据是线程私有的，因此可以在不同的线程之间独立地访问和修改，而不会相互干扰。
	// 在 Go 语言中，m 的 tls 字段是一个非常重要的字段，它与调度器和垃圾回收器密切相关，对于 Go 语言的运行时环境非常重要。由于每个 m 都有自己的 tls 字段，因此可以在不同的线程之间独立地访问和修改这些数据，从而实现了线程级别的数据隔离和并发访问。
	tls           [tlsSlots]uintptr // thread-local storage (for x86 extern register) 存储内容只对当前 线程可见 线程本地存储的是m.tls的地址，m.tls[0]存储的是当前运行的g，因此线程可以通过g找到当前m/p/g0信息
	
   ...
}
type p struct {
	...

	// 本地队列设置
    //在 Go 语言中，p 表示一个处理器（processor），它负责调度和执行 goroutine。每个 p 都有一个 runqhead、runqtail 和 runq 字段，它们用于存储当前 p 上的可运行 goroutine 队列。
	//具体来说，runqhead 和 runqtail 是两个指向 runq 数组中可运行 goroutine 的队列头和队列尾的指针。runq 数组是一个长度为 256 的数组，用于存储可运行 goroutine 的指针。当一个 goroutine 可以运行时，它会被添加到当前 p 的 runq 队列中，等待被调度器调度执行。当调度器需要选择一个 goroutine 运行时，它会从当前 p 的 runq 队列中选择一个可运行的 goroutine 运行。
	//在 Go 语言中，runqhead、runqtail 和 runq 字段是非常重要的字段，它们与调度器密切相关，对于 Go 语言的运行时环境非常重要。通过这些字段，调度器可以快速地选择一个可运行的 goroutine 运行，从而实现了高效的并发调度
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// 在 Go 语言中，p 表示一个处理器（processor），它负责调度和执行 goroutine。p 结构体中的 runnext 字段是一个指向 g（goroutine）结构体的指针，它表示下一个要运行的 goroutine。
	//具体来说，如果 runnext 不为 nil，则表示当前 goroutine 已经准备好了一个可运行的 goroutine，并且应该在当前 goroutine 的时间片还没有用完的情况下运行。这个可运行的 goroutine 将继承当前时间片中剩余的时间。如果一组 goroutine 被锁定在一个通信和等待模式中，那么这个模式将作为一个单元进行调度，从而消除了将准备好的 goroutine 添加到运行队列末尾时可能出现的（潜在的大量的）调度延迟。
    //需要注意的是，虽然其他 P 可以原子地将 runnext 设置为零，但只有所有者 P 才能将其设置为有效的 goroutine。这是因为 runnext 字段只能由当前 goroutine 来设置，其他 goroutine 不能修改它。
	runnext guintptr

 	...
}

// 全局队列
type schedt struct {
	...
	lock mutex // 通过互斥锁保证资源安全访问

	// Global runnable queue.
    // 全局队列设置
	runq     gQueue
	runqsize int32

	...
}

```

#### Q7.协程状态转换/调度类型

通常，调度指的是由 g0 按照特定策略找到下一个可执行 g 的过程.而这里的“调度”，指的是调度器 p 实现从执行一个 g 切换到另一个 g 的过程.

![9724e03e584ebe8944374d67a320d31b](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/9724e03e584ebe8944374d67a320d31b.png)

宏观调度流程

![83a6eb45a0f494f35a79dac98ecc6c41](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/83a6eb45a0f494f35a79dac98ecc6c41.png)

#### Q8.获取可执行的 g 的方法

![8ddb40b970d86c00586f7eb134f29d27](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/8ddb40b970d86c00586f7eb134f29d27.png)

>（1）p 每执行 61 次调度，会从全局队列中获取一个 goroutine 进行执行，并将一个全局队列中的 goroutine 填充到当前 p 的本地队列中.(防止协程饿死)
>
>```go
> if _p_.schedtick%61 == 0 && sched.runqsize > 0 {
>        lock(&sched.lock)
>        gp = globrunqget(_p_, 1)
>        unlock(&sched.lock)
>        if gp != nil {
>            return gp, false, false
>        }
> }
>```
>
>除了获取一个 g 用于执行外，还会额外将一个 g 从全局队列转移到 p 的本地队列，让全局队列中的 g 也得到更充分的执行机会.
>
>```go
>func globrunqget(_p_ *p, max int32) *g {
>    if sched.runqsize == 0 {
>        return nil
>    }
>
>
>    n := sched.runqsize/gomaxprocs + 1
>    if n > sched.runqsize {
>        n = sched.runqsize
>    }
>    if max > 0 && n > max {
>        n = max
>    }
>    if n > int32(len(_p_.runq))/2 {
>        n = int32(len(_p_.runq)) / 2
>    }
>
>
>    sched.runqsize -= n
>
>
>    gp := sched.runq.pop()
>    n--
>    for ; n > 0; n-- {
>        gp1 := sched.runq.pop()
>        runqput(_p_, gp1, false)
>    }
>    return gp
>```
>
>（2）尝试从 p 本地队列中获取一个可执行的 goroutine
>
>```go
> if gp, inheritTime := runqget(_p_); gp != nil {
>        return gp, inheritTime, false
>    }
>func runqget(_p_ *p) (gp *g, inheritTime bool) {
>    if next != 0 && _p_.runnext.cas(next, 0) {
>        return next.ptr(), true
>    }
>
>
>
>
>    for {
>        h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
>        t := _p_.runqtail
>        if t == h {
>            return nil, false
>        }
>        gp := _p_.runq[h%uint32(len(_p_.runq))].ptr()
>        if atomic.CasRel(&_p_.runqhead, h, h+1) { // cas-release, commits consume
>            return gp, false
>        }
>    }
>```
>
>​	I. 倘若当前 p 的 runnext 非空，直接获取即可：
>
>```go
>   if next != 0 && _p_.runnext.cas(next, 0) {
>        return next.ptr(), true
>    }
>```
>
>​	II. 加锁从 p 的本地队列中获取 g.
>
>```go
>for {
>        h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
>       // ...
>   }
>```
>
>​	需要注意，虽然本地队列是属于 p 独有的，但是由于 work-stealing 机制的存在，其他 p 可能会前来执行窃取动作，因此操作仍需加锁.
>
>​	但是，由于窃取动作发生的频率不会太高，因此当前 p 取得锁的成功率是很高的，因此可以说p 的本地队列是接近于无锁化，但没有达到真正意义的无锁.
>
>​	III. 倘若本地队列为空，直接终止并返回；
>
>```go
>     h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
>        t := _p_.runqtail
>        if t == h {
>            return nil, false
>       }
>```
>
>
>
>​	IV. 倘若本地队列存在 g，则取得队首的 g，解锁并返回.
>
>```go
>       gp := _p_.runq[h%uint32(len(_p_.runq))].ptr()
>        if atomic.CasRel(&_p_.runqhead, h, h+1) { // cas-release, commits consume
>            return gp, false
>       }
>```
>
>（3）倘若本地队列没有可执行的 g，会从全局队列中获取：(加锁，尝试并从全局队列中取队首的元素)
>
>```go
>   if sched.runqsize != 0 {
>        lock(&sched.lock)
>        gp := globrunqget(_p_, 0)
>        unlock(&sched.lock)
>        if gp != nil {
>            return gp, false, false
>        }
>    }
>```
>
>（4）倘若本地队列和全局队列都没有 g，则会获取准备就绪的网络协程：
>
>```go
> if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
>        if list := netpoll(0); !list.empty() { // non-blocking
>            gp := list.pop()
>            injectglist(&list)
>            casgstatus(gp, _Gwaiting, _Grunnable)
>            return gp, false, false
>        }
>  }
>```
>
>（5）work-stealing

#### Q9. 调度器的 work-stealing 机制

当本地队列和全局队列都没有 g，并且当前p没有可运行的 Goroutine，从其他 p 中偷取 g

```go
func stealWork(now int64) (gp *g, inheritTime bool, rnow, pollUntil int64, newWork bool) {
    pp := getg().m.p.ptr()


    ranTimer := false

//	偷取操作至多会遍历全局的 p 队列 4 次，过程中只要找到可窃取的 p 则会立即返回.

//为保证窃取行为的公平性，遍历的起点是随机的. 
    const stealTries = 4
    for i := 0; i < stealTries; i++ {
        stealTimersOrRunNextG := i == stealTries-1


        for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
            // ...
        }
    }


    return nil, false, now, pollUntil, ranTime
```

```go
func runqgrab(_p_ *p, batch *[256]guintptr, batchHead uint32, stealRunNextG bool) uint32 {
    for {
        // 每次对一个 p 尝试窃取前，会对其局部队列加锁；
        h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
        t := atomic.LoadAcq(&_p_.runqtail) // load-acquire, synchronize with the producer
        //尝试偷取其现有的一半 g，并且返回实际偷取的数量.
        n := t - h
        n = n - n/2
        if n == 0 {
            if stealRunNextG {
                // Try to steal from _p_.runnext.
                if next := _p_.runnext; next != 0 {
                    if _p_.status == _Prunning {
                        
                        if GOOS != "windows" && GOOS != "openbsd" && GOOS != "netbsd" {
                            usleep(3)
                        } else {
                            osyield()
                        }
                    }
                    if !_p_.runnext.cas(next, 0) {
                        continue
                    }
                    batch[batchHead%uint32(len(batch))] = next
                    return 1
                }
            }
            return 0
        }
        if n > uint32(len(_p_.runq)/2) { // read inconsistent h and t
            continue
        }
        for i := uint32(0); i < n; i++ {
            g := _p_.runq[(h+i)%uint32(len(_p_.runq))]
            batch[(batchHead+i)%uint32(len(batch))] = g
        }
        if atomic.CasRel(&_p_.runqhead, h, h+n) { // cas-release, commits consume
            return n
        }
    }
}
```

![image-20230715231941751](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230715231941751.png)



#### Q10. 执行 goroutine 逻辑

![b7700a40d929858747093d7775904d43](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/b7700a40d929858747093d7775904d43.png)

>（1）更新 g 的状态信息，建立 g 与 m 之间的绑定关系；
>
>（2）更新 p 的总调度次数；
>
>（3）调用 gogo 方法，执行 goroutine 中的任务.
>
>```go
>func execute(gp *g, inheritTime bool) {
>    _g_ := getg()
>
>
>    _g_.m.curg = gp
>    gp.m = _g_.m
>    casgstatus(gp, _Grunnable, _Grunning)
>    gp.waitsince = 0
>    gp.preempt = false
>    gp.stackguard0 = gp.stack.lo + _StackGuard
>    if !inheritTime {
>        _g_.m.p.ptr().schedtick++
>    }
>
>
>    gogo(&gp.sched) 
>```
>
>

#### Q11. 主动让渡与被动阻塞/调度结束与抢占调度

g 执行主动让渡时，会调用 mcall 方法将执行权归还给 g0，并由 g0 调用 gosched_m 方法进行解绑/状态转换等操作

![ccb22d725b5da191eae03aea41caee56](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/ccb22d725b5da191eae03aea41caee56.png)

```go
func Gosched() {
    // ...
    mcall(gosched_m)
}
func gosched_m(gp *g) {
    goschedImpl(gp)
}


func goschedImpl(gp *g) {
    status := readgstatus(gp)
    if status&^_Gscan != _Grunning {
        dumpgstatus(gp)
        throw("bad g status")
    }
    //将当前 g 的状态由执行中切换为待执行 _Grunnable：
    casgstatus(gp, _Grunning, _Grunnable) 
    //调用 dropg() 方法，将当前的 m 和 g 解绑；
    dropg()
    lock(&sched.lock)
    //将 g 添加到全局队列当中
    globrunqput(gp)
    unlock(&sched.lock)
	//开启新一轮的调度
    schedule()
```

g 需要被动调度时，会调用 mcall 方法切换至 g0，并调用 park_m 方法将 g 置为阻塞态

![图片](https://mmbiz.qpic.cn/mmbiz_png/3ic3aBqT2ibZv7BArqKhicntmW5bZrgickiaguTPjUX0icRicMBwF5uwbylRHAxItjK05Ml5vZo4icYhGVS8hgBsiaylHyw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

```go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
    // ...
    mcall(park_m)
}

func park_m(gp *g) {
    _g_ := getg()

	//（1）将当前 g 的状态由 running 改为 waiting；
    casgstatus(gp, _Grunning, _Gwaiting)
    // （2）将 g 与 m 解绑；
    dropg()


    // （3）执行新一轮的调度 schedule.
    schedule()
```

当因被动调度陷入阻塞态的 g 需要被唤醒时，会由其他 g 负责将 g 的状态由 waiting 改为 runnable，然后会将其添加到唤醒者的 p 的本地队列中

```go
func goready(gp *g, traceskip int) {
    systemstack(func() {
        ready(gp, traceskip, true)
    })
}

func ready(gp *g, traceskip int, next bool) {
    // ...
    _g_ := getg()
    // （1）先将 g 的状态从阻塞态改为可执行的状态；
    casgstatus(gp, _Gwaiting, _Grunnable)
    // （2）调用 runqput 将当前 g 添加到唤醒者 p 的本地队列中，如果队列满了，会连带 g 一起将一半的元素转移到全局队列.
    runqput(_g_.m.p.ptr(), gp, next)
    // ...
}
```

当 g 执行完成时，会先执行 mcall 方法切换至 g0，然后调用 goexit0 方法,以结束自己的生命周期

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/20230716010433.png)



```go
func goexit1() {
    // ...
    mcall(goexit0)
}
func goexit0(gp *g) {
    _g_ := getg()
    _p_ := _g_.m.p.ptr()

	//（1）将 g 状态置为 dead；
    casgstatus(gp, _Grunning, _Gdead)
    // ...
    gp.m = nil
    // ...

	//（2）解绑 g 和 m；
    dropg()


    // （3）开启新一轮的调度.
    schedule()
```



####  Q12. 两种g的转换

goroutine 的类型可分为两类：

I.  负责调度普通 g 的 g0，执行固定的调度流程，与 m 的关系为一对一；

II.  负责执行用户函数的普通 g.

m 通过 p 调度执行的 goroutine 永远在普通 g 和 g0 之间进行切换，当 g0 找到可执行的 g 时，会调用 gogo 方法，调度 g 执行用户定义的任务；当 g 需要主动让渡或被动调度时，会触发 mcall 方法，将执行权重新交还给 g0.



gogo 和 mcall 可以理解为对偶关系

#### Q13. 系统调用前后

在 m 需要执行系统调用前，会先执行位于 runtime/proc.go 的 reentersyscall 的方法

```go
//这段代码是 Go 语言中的系统调用处理函数 reentersyscall，它用于将当前 goroutine 从用户态切换到内核态，执行系统调用,并将当前 goroutine 所在的 P 的状态修改为 _Psyscall。

1. 获取当前 goroutine 的 G 结构体指针 _g_。

2. 调用 save 函数保存当前 goroutine 的程序计数器 pc 和栈指针 sp。

3. 将当前 goroutine 的 syscallpc 和 syscallsp 字段设置为 pc 和 sp。

4. 将当前 goroutine 的状态从 _Grunning（运行状态）修改为 _Gsyscall（系统调用状态）。

5. 获取当前 goroutine 所在的 P 结构体指针 pp。

6. 将当前 goroutine 所在的 M 结构体的 p 字段设置为 0，表示当前 goroutine 不再运行在任何一个 P 上。

7. 将当前 goroutine 所在的 P 结构体的 m 字段设置为 0，表示当前 P 不再运行任何一个 goroutine。

8. 将当前 P 的状态从 _Prunning（运行状态）修改为 _Psyscall（系统调用状态）。
func reentersyscall(pc, sp uintptr) {
    _g_ := getg()


    // ...
    save(pc, sp)
    _g_.syscallsp = sp
    _g_.syscallpc = pc
    casgstatus(_g_, _Grunning, _Gsyscall)
    // ...


    pp := _g_.m.p.ptr()
    pp.m = 0
    _g_.m.oldp.set(pp)
    _g_.m.p = 0
    atomic.Store(&pp.status, _Psyscall)
    // ...
```

当 m 完成了内核态的系统调用之后，此时会步入位于 runtime/proc.go 的 exitsyscall 函数中，尝试寻找 p 重新开始运作：

```go
这段代码是 Go 语言中的系统调用处理函数 exitsyscall，它用于将当前 goroutine 从内核态切换回用户态，结束系统调用，并重新调度 goroutine。

1. 获取当前 goroutine 的 G 结构体指针 _g_。

2. 调用 exitsyscallfast 函数尝试快速结束系统调用。如果快速结束成功，则将当前 goroutine 的状态从 _Gsyscall（系统调用状态）修改为 _Grunning（运行状态），并直接返回。

3. 如果快速结束失败，则调用 mcall 函数，将当前 goroutine 所在的 M 结构体的 curg 字段设置为 _g_，然后调用 exitsyscall0 函数结束系统调用。

4. 在 exitsyscall0 函数中，将当前 goroutine 的状态从 _Gsyscall（系统调用状态）修改为 _Grunning（运行状态），然后调用 schedule 函数重新调度 goroutine。

func exitsyscall() {
    _g_ := getg()
    
    // ...
    if exitsyscallfast(oldp) {
        // ...
        casgstatus(_g_, _Gsyscall, _Grunning)
        // ...
        return
    }


    // ...
    mcall(exitsyscall0)
    // ...
}
这段代码是 Go 语言中的系统调用处理函数 exitsyscall0，它用于结束系统调用并重新调度 goroutine，以便让其他 goroutine 能够充分利用 CPU 资源。


1. 将当前 goroutine 的状态从 _Gsyscall（系统调用状态）修改为 _Grunnable（可运行状态）。

2. 调用 dropg 函数将当前 goroutine 从当前 M 的 curg 字段中移除以并解绑 g 和 m 的关系：

3. 获取调度器的全局锁，并尝试获取一个空闲的 P。

4. 如果成功获取到一个空闲的 P，则将当前 goroutine 添加到该 P 的本地队列中，并调用 execute 函数执行该 goroutine。由于 execute 函数永远不会返回，因此这个 goroutine 将一直运行在该 P 上。

5. 如果没有获取到空闲的 P，则将当前 goroutine 添加到全局队列中，等待其他 P 的工作窃取。

6. 释放调度器的全局锁。

7. 调用 stopm 函数将当前 M 的状态设置为 _Mdead（死亡状态）。

8. 调用 schedule 函数重新调度 goroutine。由于 schedule 函数永远不会返回，因此这个 goroutine 将一直运行在某个 P 上。

func exitsyscall0(gp *g) {
    casgstatus(gp, _Gsyscall, _Grunnable)
    dropg()
    lock(&sched.lock)
    var _p_ *p
    if schedEnabled(gp) {
        _p_, _ = pidleget(0)
    }
    
    var locked bool
    if _p_ == nil {
        globrunqput(gp)
    } 
    
    unlock(&sched.lock)
    if _p_ != nil {
        acquirep(_p_)
        execute(gp, false) // Never returns.
    }
    
    // ...
    
    stopm()
    schedule() // Never returns.
}
```



#### Q14. Goroutine 调度器的 GMP 模型的设计思想

![image-20230809224740269](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809224740269.png)

##### Q14.01 GMP 模型

Processor，它包含了运行 goroutine 的资源，如果线程想运行 goroutine，必须先获取 P，P 中还包含了可运行的 G 队列。
在 Go 中，线程是运行 goroutine 的实体，调度器的功能是把可运行的 goroutine 分配到工作线程上

![image-20230809224922005](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809224922005.png)

- 全局队列（Global Queue）：存放等待运行的 G（协程）。
- P 的本地队列：同全局队列类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。新建 G时，G优先加入到 P 的本地队列，如果队列满了，则会把本地队列中一半的 G 移动到全局队列。
- P 列表：所有的 P 都在程序启动时创建，并保存在数组中，最多有 GOMAXPROCS(可配置) 个。
- M：线程想运行任务就得获取 P，从 P 的本地队列获取 G，P 队列为空时，M 也会尝试从全局队列拿一批 G 放到 P 的本地队列，或从其他 P 的本地队列偷一半放到自己 P 的本地队列。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。

**Goroutine 调度器和 OS 调度器是通过 M 结合起来的，每个 M 都代表了 1 个内核线程，OS 调度器负责把内核线程分配到 CPU 的核上执行**

##### Q14.02 有关 P 和 M 的个数问题
1. P 的数量
    由启动时环境变量 $GOMAXPROCS 或者是由 runtime 的方法 GOMAXPROCS() 决定。在Go1.5之后GOMAXPROCS被默认设置可用的核数，而之前则默认为1。这意味着在程序执行的任意时刻都只有 $GOMAXPROCS 个 goroutine 在同时运行

2. M 的数量

   - go 语言本身的限制：go 程序启动时，会设置 M 的最大数量，默认 10000. 但是内核很难支持这么多的线程数，所以这个限制可以忽略

   - runtime/debug 中的 SetMaxThreads 函数，设置 M 的最大数量

   - 一个 M 阻塞了，会创建新的 M

**M 与 P 的数量没有绝对关系，一个 M 阻塞，P 就会去创建或者切换另一个 M，所以，即使 P 的默认数量是 1，也有可能会创建很多个 M 出来**

##### Q14.03 有关 P 和 M 的创建时间问题
- P 何时创建：在确定了 P 的最大数量 n 后，运行时系统会根据这个数量创建 n 个 P。
- M 何时创建：没有足够的 M 来关联 P 并运行其中的可运行的 G。比如所有的 M 此时都阻塞住了，而 P 中还有很多就绪任务，就会去寻找空闲的 M，而没有空闲的，就会去创建新的 M。

##### Q14.04 调度器的设计策略

**复用线程：避免频繁的创建、销毁线程，而是对线程的复用**

1. **work stealing 机制**

    **当本线程无可运行的 G 时，尝试从其他线程绑定的 P 偷取 G，而不是销毁线程**

2. **hand off 机制**

    **当本线程因为 G 进行系统调用阻塞时，线程释放绑定的 P，把 P 转移给其他空闲的线程执行**

利用并行：GOMAXPROCS 设置 P 的数量，最多有 GOMAXPROCS 个线程分布在多个 CPU 上同时运行。GOMAXPROCS 也限制了并发的程度
抢占：在coroutine 中要等待一个协程主动让出 CPU 才执行下一个协程，在 Go 中，一个 goroutine 最多占用 CPU 10ms，防止其他 goroutine 被饿死，这就是 goroutine 不同于 coroutine 的一个地方
全局 G 队列：在新的调度器中依然有全局 G 队列，但功能已经被弱化了，当 M 执行 work stealing 从其他 P 偷不到 G 时，它可以从全局 G 队列获取 G

##### Q14.05 go func () 调度流程

![image-20230809225729278](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809225729278.png)

**从上图可以分析出几个结论：**

1. 通过 go func () 来创建一个 goroutine；

2. 有两个存储 G 的队列，一个是局部调度器 P 的本地队列、一个是全局 G 队列。新创建的 G 会先保存在 P 的本地队列中，如果 P 的本地队列已经满了就会保存在全局的队列中；

3. G 只能运行在 M 中，一个 M 必须持有一个 P，M 与 P 是 1：1 的关系。M 会从 P 的本地队列弹出一个可执行状态的 G 来执行，如果 P 的本地队列为空，就会从其他的 MP 组合偷取一个可执行的 G 来执行；

4. 一个 M 调度 G 执行的过程是一个循环机制；

5. 当 M 执行某一个 G 时候如果发生了 syscall 或则其余阻塞操作，M 会阻塞，如果当前有一些 G 在执行，runtime 会把这个线程 M 从 P 中摘除 (detach)，然后再创建一个新的操作系统的线程 (如果有空闲的线程可用就复用空闲线程) 来服务于这个 P；

6. 当 M 系统调用结束时候，这个 G 会尝试获取一个空闲的 P 执行，并放入到这个 P 的本地队列。如果获取不到 P，那么这个线程 M 变成休眠状态， 加入到空闲线程中，然后这个 G 会被放入全局队列中。

##### Q14.03 调度器的生命周期

![image-20230809230025599](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809230025599.png)

**特殊的 M0 和 G0**

>M0：M0 是启动程序后的编号为 0 的主线程，这个 M 对应的实例会在全局变量 runtime.m0 中，不需要在 heap 上分配，M0 负责执行初始化操作和启动第一个 G， 在之后 M0 就和其他的 M 一样了。
>
>G0：G0 是每次启动一个 M 都会第一个创建的 gourtine，G0 仅用于负责调度的 G，G0 不指向任何可执行的函数，每个 M 都会有一个自己的 G0。在调度或系统调用时会使用 G0 的栈空间，全局变量的 G0 是 M0 的 G0。

调度器的生命周期几乎占满了一个 Go 程序的一生，runtime.main 的 goroutine 执行之前都是为调度器做准备工作，runtime.main 的 goroutine 运行，才是调度器的真正开始，直到 runtime.main 结束而结束

#### Q15. Go 调度器调度场景过程全解析
- 场景1
P 拥有 G1，M1 获取 P 后开始运行 G1，G1 使用 go func() 创建了 G2，为了局部性 G2 优先加入到 P1 的本地队列

![image-20230809232110096](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809232110096.png)

- 场景2 

  G1 运行完成后 (函数：goexit)，M 上运行的 goroutine 切换为 G0，G0 负责调度时协程的切换函数：schedule）。从 P 的本地队列取 G2，从 G0 切换到 G2，并开始运行 G2 (函数：execute)。实现了线程 M1 的复用

  ![image-20230809232844396](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809232844396.png)

- 场景3 

  假设每个 P 的本地队列只能存 4 个 G。G2 要创建了 6 个 G，前 3 个 G（G3, G4, G5,G6）已经加入 p1 的本地队列，p1 本地队列满了

  ![image-20230809233720761](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809233720761.png)

- 场景4

  G2 在创建 G7 的时候，发现 P1 的本地队列已满，需要执行负载均衡 (把 P1 中本地队列中前一半的 G，还有新创建 G 转移到全局队列)

  ![image-20230809234550882](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809234550882.png)

- 场景5

  G2 创建 G8 时，P1 的本地队列未满，所以 G8 会被加入到 P1 的本地队列
  加入到 P1 点本地队列的原因还是因为 P1 此时在与 M1 绑定，而 G2 此时是 M1 在执行。所以 G2 创建的新的 G 会优先放置到自己的 M 绑定的 P 上
  
  ![image-20230809235149315](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809235149315.png)

- 场景6

  **规定：在创建 G 时，运行的 G 会尝试唤醒其他空闲的 P 和 M 组合去执行**

  **假定 G2 唤醒了 M2，M2 绑定了 P2，并运行 G0，**但 P2 本地队列没有 G，M2 此时为自旋线程（没有 G 但为运行状态的线程，不断寻找 G)

  ![image-20230810001535957](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230810001535957.png)

- 场景7

  **M2 尝试从全局队列 (简称 “GQ”) 取一批 G 放到 P2 的本地队列（函数：findrunnable()）。M2 从全局队列取的 G 数量符合下面的公式:**

  >> n = min(len(GQ)/GOMAXPROCS + 1, len(GQ/2))

  **至少从全局队列取 1 个 g，但每次不要从全局队列移动太多的 g 到 p 本地队列，给其他 p 留点。这是从全局队列到 P 本地队列的负载均衡**

  ![image-20230810002138108](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230810002138108.png)

  **假定我们场景中一共有 4 个 P（GOMAXPROCS 设置为 4，那么我们允许最多就能用 4 个 P 来供 M 使用）。所以 M2 只从能从全局队列取 1 个 G（即 G3）移动 P2 本地队列，然后完成从 G0 到 G3 的切换，运行 G3**

- 场景8

  **假设 G2 一直在 M1 上运行，经过 2 轮后，M2 已经把 G7、G4 从全局队列获取到了 P2 的本地队列并完成运行，全局队列和 P2 的本地队列都空了，如 图的左半部分**

  ![image-20230810002531646](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230810002531646.png)

  全局队列已经没有 G，那 m 就要执行 work stealing (偷取)：从其他有 G 的 P 哪里偷取一半 G 过来，放到自己的 P 本地队列。P2 从 P1 的本地队列尾部取一半的 G，本例中一半则只有 1 个 G8，放到 P2 的本地队列并执行

- 场景9

  G1 本地队列 G5、G6 已经被其他 M 偷走并运行完成，当前 M1 和 M2 分别在运行 G2 和 G8，M3 和 M4 没有 goroutine 可以运行，M3 和 M4 处于自旋状态，它们不断寻找 goroutine

  ![image-20230810002933581](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230810002933581.png)

  为什么要让 m3 和 m4 自旋，自旋本质是在运行，线程在运行却没有执行 G，就变成了浪费 CPU. 为什么不销毁现场，来节约 CPU 资源。因为创建和销毁 CPU 也会浪费时间，我们希望当有新 goroutine 创建时，立刻能有 M 运行它，如果销毁再新建就增加了时延，降低了效率。当然也考虑了过多的自旋线程是浪费 CPU，所以系统中最多有 GOMAXPROCS 个自旋的线程 (当前例子中的 GOMAXPROCS=4，所以一共 4 个 P)，多余的没事做线程会让他们休眠。

- 场景10

  假定当前除了 M3 和 M4 为自旋线程，还有 M5 和 M6 为空闲的线程 (没有得到 P 的绑定，注意我们这里最多就只能够存在 4 个 P，所以 P 的数量应该永远是 M>=P, 大部分都是 M 在抢占需要运行的 P)，G8 创建了 G9，G8 进行了阻塞的系统调用，M2 和 P2 立即解绑，P2 会执行以下判断：如果 P2 本地队列有 G、全局队列有 G 或有空闲的 M，P2 都会立马唤醒 1 个 M 和它绑定，否则 P2 则会加入到空闲 P 列表，等待 M 来获取可用的 p。本场景中，P2 本地队列有 G9，可以和其他空闲的线程 M5 绑定。

  ![image-20230810004220843](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230810004220843.png)

  

- 场景11

  **G8 创建了 G9，假如 G8 进行了非阻塞系统调用。**

![image-20230810121253818](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230810121253818.png)

M1 和 P1 会解绑，但 M1 会记住 P1，然后 G8 和 M1 进入系统调用状态。当 G8 和 M1 退出系统调用时，会尝试获取 P1，如果无法获取，则获取空闲的 P，如果依然没有，G8 会被记为可运行状态，并加入到全局队列，M1 因为没有 P 的绑定而变成休眠状态 (长时间休眠等待 GC 回收销毁)。

