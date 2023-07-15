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

#### Q3. 三种模型对比

| 模型      | 弱依赖内核 | 可并行 | 可应对阻塞 | 栈可动态扩缩 |
| --------- | ---------- | ------ | ---------- | ------------ |
| 线程      | N          | Y      | Y          | N            |
| 协程      | Y          | N      | N          | N            |
| goroutine | Y          | Y      | Y          | Y            |

#### Q4. gmp 理论模型

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

#### Q5. gmp 数据结构

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
	id          int32
	status      uint32 // one of pidle/prunning/...
	link        puintptr
	schedtick   uint32     // incremented on every scheduler call
	syscalltick uint32     // incremented on every system call
	sysmontick  sysmontick // last tick observed by sysmon
	m           muintptr   // back-link to associated m (nil if idle)
	mcache      *mcache
	pcache      pageCache
	raceprocctx uintptr

	deferpool    []*_defer // pool of available defer structs (see panic.go)
	deferpoolbuf [32]*_defer

	// Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
	goidcache    uint64
	goidcacheend uint64

	// Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// runnext, if non-nil, is a runnable G that was ready'd by
	// the current G and should be run next instead of what's in
	// runq if there's time remaining in the running G's time
	// slice. It will inherit the time left in the current time
	// slice. If a set of goroutines is locked in a
	// communicate-and-wait pattern, this schedules that set as a
	// unit and eliminates the (potentially large) scheduling
	// latency that otherwise arises from adding the ready'd
	// goroutines to the end of the run queue.
	//
	// Note that while other P's may atomically CAS this to zero,
	// only the owner P can CAS it to a valid G.
	runnext guintptr

	// Available G's (status == Gdead)
	gFree struct {
		gList
		n int32
	}

	sudogcache []*sudog
	sudogbuf   [128]*sudog

	// Cache of mspan objects from the heap.
	mspancache struct {
		// We need an explicit length here because this field is used
		// in allocation codepaths where write barriers are not allowed,
		// and eliminating the write barrier/keeping it eliminated from
		// slice updates is tricky, moreso than just managing the length
		// ourselves.
		len int
		buf [128]*mspan
	}

	tracebuf traceBufPtr

	// traceSweep indicates the sweep events should be traced.
	// This is used to defer the sweep start event until a span
	// has actually been swept.
	traceSweep bool
	// traceSwept and traceReclaimed track the number of bytes
	// swept and reclaimed by sweeping in the current sweep loop.
	traceSwept, traceReclaimed uintptr

	palloc persistentAlloc // per-P to avoid mutex

	// The when field of the first entry on the timer heap.
	// This is 0 if the timer heap is empty.
	timer0When atomic.Int64

	// The earliest known nextwhen field of a timer with
	// timerModifiedEarlier status. Because the timer may have been
	// modified again, there need not be any timer with this value.
	// This is 0 if there are no timerModifiedEarlier timers.
	timerModifiedEarliest atomic.Int64

	// Per-P GC state
	gcAssistTime         int64 // Nanoseconds in assistAlloc
	gcFractionalMarkTime int64 // Nanoseconds in fractional mark worker (atomic)

	// limiterEvent tracks events for the GC CPU limiter.
	limiterEvent limiterEvent

	// gcMarkWorkerMode is the mode for the next mark worker to run in.
	// That is, this is used to communicate with the worker goroutine
	// selected for immediate execution by
	// gcController.findRunnableGCWorker. When scheduling other goroutines,
	// this field must be set to gcMarkWorkerNotWorker.
	gcMarkWorkerMode gcMarkWorkerMode
	// gcMarkWorkerStartTime is the nanotime() at which the most recent
	// mark worker started.
	gcMarkWorkerStartTime int64

	// gcw is this P's GC work buffer cache. The work buffer is
	// filled by write barriers, drained by mutator assists, and
	// disposed on certain GC state transitions.
	gcw gcWork

	// wbBuf is this P's GC write barrier buffer.
	//
	// TODO: Consider caching this in the running G.
	wbBuf wbBuf

	runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

	// statsSeq is a counter indicating whether this P is currently
	// writing any stats. Its value is even when not, odd when it is.
	statsSeq atomic.Uint32

	// Lock for timers. We normally access the timers while running
	// on this P, but the scheduler can also do it from a different P.
	timersLock mutex

	// Actions to take at some time. This is used to implement the
	// standard library's time package.
	// Must hold timersLock to access.
	timers []*timer

	// Number of timers in P's heap.
	numTimers atomic.Uint32

	// Number of timerDeleted timers in P's heap.
	deletedTimers atomic.Uint32

	// Race context used while executing timer functions.
	timerRaceCtx uintptr

	// maxStackScanDelta accumulates the amount of stack space held by
	// live goroutines (i.e. those eligible for stack scanning).
	// Flushed to gcController.maxStackScan once maxStackScanSlack
	// or -maxStackScanSlack is reached.
	maxStackScanDelta int64

	// gc-time statistics about current goroutines
	// Note that this differs from maxStackScan in that this
	// accumulates the actual stack observed to be used at GC time (hi - sp),
	// not an instantaneous measure of the total stack size that might need
	// to be scanned (hi - lo).
	scannedStackSize uint64 // stack size of goroutines scanned by this P
	scannedStacks    uint64 // number of goroutines scanned by this P

	// preempt is set to indicate that this P should be enter the
	// scheduler ASAP (regardless of what G is running on it).
	preempt bool

	// pageTraceBuf is a buffer for writing out page allocation/free/scavenge traces.
	//
	// Used only if GOEXPERIMENT=pagetrace.
	pageTraceBuf pageTraceBuf

	// Padding is no longer needed. False sharing is now not a worry because p is large enough
	// that its size class is an integer multiple of the cache line size (for any of our architectures).
}
```



#### Q6.协程状态转换/调度类型



#### Q7.获取可执行的 g 的方法



#### Q8. 调度器的 work-stealing 机制



#### Q9. 执行 goroutine 逻辑



#### Q10. 主动让渡与被动阻塞/调度结束与抢占调度









