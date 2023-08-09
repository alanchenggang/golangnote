#### Q1. Go 逃逸分析

##### Q1.01 什么是逃逸分析

在编译程序优化理论中，逃逸分析是一种确定指针动态范围的方法，简单来说就是分析在程序的哪些地方可以访问到该指针。

Go是通过在编译器里做逃逸分析（escape analysis）来决定一个对象放栈上还是放堆上，不逃逸的对象放栈上，可能逃逸的放堆上；即我发现`变量`在退出函数后没有用了，那么就把丢到栈上，毕竟栈上的内存分配和回收比堆上快很多；反之，函数内的普通变量经过`逃逸分析`后，发现在函数退出后`变量`还有在其他地方上引用，那就将`变量`分配在堆上。做到按需分配

##### Q1.02 堆和栈

要理解什么是逃逸分析会涉及堆和栈的一些基本知识
- 堆（Heap）：一般来讲是人为手动进行管理，手动申请、分配、释放。堆适合不可预知大小的内存分配，这也意味着为此付出的代价是分配速度较慢，而且会形成内存碎片
- 栈（Stack）：由编译器进行管理，自动申请、分配、释放。一般不会太大，因此栈的分配和回收速度非常快；我们常见的函数参数（不同平台允许存放的数量不同），局部变量等都会存放在栈上

栈分配内存只需要两个CPU指令：“PUSH”和“RELEASE”，分配和释放；而堆分配内存首先需要去找到一块大小合适的内存块，之后要通过垃圾回收才能释放。

##### Q1.03 为何需要逃逸分析

1. 减少`gc`压力，栈上的变量，随着函数退出后系统直接回收，不需要`gc`标记后再清除
2. 减少内存碎片的产生
3. 减轻分配堆内存的开销，提高程序的运行速度

##### Q1.04 如何确定是否逃逸
在Go中通过逃逸分析日志来确定变量是否逃逸，开启逃逸分析日志：
```sh
go run -gcflags '-m -l' main.go
```
>-m 会打印出逃逸分析的优化策略，实际上最多总共可以用 4 个 -m，但是信息量较大，一般用 1 个就可以了
>-l 会禁用函数内联，在这里禁用掉内联能更好的观察逃逸情况，减少干扰


##### Q1.05 逃逸案例
1. 取地址发生逃逸

   ```go
   package main
   ​
   type UserData struct {
     Name  string
   }
   ​
   func main() {
     var info UserData
     info.Name = "WilburXu"
     _ = GetUserInfo(info)
   }
   
   func GetUserInfo(userInfo UserData) *UserData {
     return &userInfo
   }
   ```

   ![image-20230809221929583](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809221929583.png)

   >GetUserInfo函数里面的变量 `userInfo` 逃到堆上了（分配到堆内存空间上了）。
   >
   >GetUserInfo 函数的返回值为 \*UserData 指针类型，然后 将值变量`userInfo` 的地址返回，此时编译器会判断该值可能会在函数外使用，就将其分配到了堆上，所以变量`userInfo`就逃逸了。

优化方案

```go
func main() {
  var info UserData
  info.Name = "WilburXu"
  _ = GetUserInfo(&info)
}

func GetUserInfo(userInfo *UserData) *UserData {
  return userInfo
}
```

![image-20230809222435620](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809222435620.png)

对一个变量取地址，可能会被分配到堆上。但是编译器进行逃逸分析后，如果发现到在函数返回后，此变量不会被引用，那么还是会被分配到栈上

2. 未确定类型逃逸

```go
package main
​
type User struct {
  name interface{}
}
​
func main() {
  name := "WilburXu"
  MyPrintln(name)
}

func MyPrintln(one interface{}) (n int, err error) {
  var userInfo = new(User)
  userInfo.name = one // 泛型赋值 逃逸咯
  return
}
```

![image-20230809222604446](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809222604446.png)



对于`interface`类型，很遗憾，编译器在编译的时候很难知道在函数的调用或者结构体的赋值过程会是怎么类型，因此只能分配到`堆`上。

优化方案
明确结构体user的成员变量name的类型

##### Q1.06 总结
不要盲目使用变量的指针作为函数参数，虽然它会减少复制操作。但其实当参数为变量自身的时候，复制是在栈上完成的操作，开销远比变量逃逸后动态地在堆上分配内存少的多。

Go的编译器就如一个聪明的孩子一般，大多时候在逃逸分析问题上的处理都令人眼前一亮，但有时闹性子的时候处理也是非常粗糙的分析或完全放弃，  所以也需要我们在编写代码的时候多多观察，多多留意了。

#### Q2.Golang的GMP原理与调度

##### Q2.01 Golang调度器的由来

1.  单进程时代不需要调度器
2.  多进程 / 线程时代有了调度器需求
3.  协程来提高 CPU 利用率

##### Q2.02 线程和协程绑定的三种方式

1. N 个协程绑定 1 个线程
2. 1 个协程绑定 1 个线程
3. M 个协程绑定 1 个线程

##### Q2.03 Go 语言的协程 goroutine
Go 为了提供更容易使用的并发方法，使用了 goroutine 和 channel。goroutine 来自协程的概念，让一组可复用的函数运行在一组线程之上，即使有协程阻塞，该线程的其他协程也可以被 runtime 调度，转移到其他可运行的线程上。最关键的是，程序员看不到这些底层的细节，这就降低了编程的难度，提供了更容易的并发。

Go 中，协程被称为 goroutine，它非常轻量，一个 goroutine 只占几 KB，并且这几 KB 就足够 goroutine 运行完，这就能在有限的内存空间内支持大量 goroutine，支持了更多的并发。虽然一个 goroutine 的栈只占几 KB，但实际是可伸缩的，如果需要更多内容，runtime 会自动为 goroutine 分配。

Goroutine 特点：

- 占用内存更小（几 kb）
- 调度更灵活 (runtime 调度)

#### Q3. Goroutine 调度器的 GMP 模型的设计思想

![image-20230809224740269](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809224740269.png)

##### Q3.01 GMP 模型

Processor，它包含了运行 goroutine 的资源，如果线程想运行 goroutine，必须先获取 P，P 中还包含了可运行的 G 队列。
在 Go 中，线程是运行 goroutine 的实体，调度器的功能是把可运行的 goroutine 分配到工作线程上

![image-20230809224922005](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809224922005.png)

- 全局队列（Global Queue）：存放等待运行的 G（协程）。
- P 的本地队列：同全局队列类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。新建 G时，G优先加入到 P 的本地队列，如果队列满了，则会把本地队列中一半的 G 移动到全局队列。
- P 列表：所有的 P 都在程序启动时创建，并保存在数组中，最多有 GOMAXPROCS(可配置) 个。
- M：线程想运行任务就得获取 P，从 P 的本地队列获取 G，P 队列为空时，M 也会尝试从全局队列拿一批 G 放到 P 的本地队列，或从其他 P 的本地队列偷一半放到自己 P 的本地队列。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。

**Goroutine 调度器和 OS 调度器是通过 M 结合起来的，每个 M 都代表了 1 个内核线程，OS 调度器负责把内核线程分配到 CPU 的核上执行**

##### Q3.02 有关 P 和 M 的个数问题
1. P 的数量
    由启动时环境变量 $GOMAXPROCS 或者是由 runtime 的方法 GOMAXPROCS() 决定。在Go1.5之后GOMAXPROCS被默认设置可用的核数，而之前则默认为1。这意味着在程序执行的任意时刻都只有 $GOMAXPROCS 个 goroutine 在同时运行

2. M 的数量

   - go 语言本身的限制：go 程序启动时，会设置 M 的最大数量，默认 10000. 但是内核很难支持这么多的线程数，所以这个限制可以忽略

   - runtime/debug 中的 SetMaxThreads 函数，设置 M 的最大数量

   - 一个 M 阻塞了，会创建新的 M

**M 与 P 的数量没有绝对关系，一个 M 阻塞，P 就会去创建或者切换另一个 M，所以，即使 P 的默认数量是 1，也有可能会创建很多个 M 出来**

##### Q3.03 有关 P 和 M 的创建时间问题
- P 何时创建：在确定了 P 的最大数量 n 后，运行时系统会根据这个数量创建 n 个 P。
- M 何时创建：没有足够的 M 来关联 P 并运行其中的可运行的 G。比如所有的 M 此时都阻塞住了，而 P 中还有很多就绪任务，就会去寻找空闲的 M，而没有空闲的，就会去创建新的 M。

##### Q3.04 调度器的设计策略

**复用线程：避免频繁的创建、销毁线程，而是对线程的复用**

1. **work stealing 机制**

    **当本线程无可运行的 G 时，尝试从其他线程绑定的 P 偷取 G，而不是销毁线程**

2. **hand off 机制**

    **当本线程因为 G 进行系统调用阻塞时，线程释放绑定的 P，把 P 转移给其他空闲的线程执行**

利用并行：GOMAXPROCS 设置 P 的数量，最多有 GOMAXPROCS 个线程分布在多个 CPU 上同时运行。GOMAXPROCS 也限制了并发的程度
抢占：在coroutine 中要等待一个协程主动让出 CPU 才执行下一个协程，在 Go 中，一个 goroutine 最多占用 CPU 10ms，防止其他 goroutine 被饿死，这就是 goroutine 不同于 coroutine 的一个地方
全局 G 队列：在新的调度器中依然有全局 G 队列，但功能已经被弱化了，当 M 执行 work stealing 从其他 P 偷不到 G 时，它可以从全局 G 队列获取 G

##### Q3.05 go func () 调度流程

![image-20230809225729278](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809225729278.png)

**从上图可以分析出几个结论：**

1. 通过 go func () 来创建一个 goroutine；

2. 有两个存储 G 的队列，一个是局部调度器 P 的本地队列、一个是全局 G 队列。新创建的 G 会先保存在 P 的本地队列中，如果 P 的本地队列已经满了就会保存在全局的队列中；

3. G 只能运行在 M 中，一个 M 必须持有一个 P，M 与 P 是 1：1 的关系。M 会从 P 的本地队列弹出一个可执行状态的 G 来执行，如果 P 的本地队列为空，就会从其他的 MP 组合偷取一个可执行的 G 来执行；

4. 一个 M 调度 G 执行的过程是一个循环机制；

5. 当 M 执行某一个 G 时候如果发生了 syscall 或则其余阻塞操作，M 会阻塞，如果当前有一些 G 在执行，runtime 会把这个线程 M 从 P 中摘除 (detach)，然后再创建一个新的操作系统的线程 (如果有空闲的线程可用就复用空闲线程) 来服务于这个 P；

6. 当 M 系统调用结束时候，这个 G 会尝试获取一个空闲的 P 执行，并放入到这个 P 的本地队列。如果获取不到 P，那么这个线程 M 变成休眠状态， 加入到空闲线程中，然后这个 G 会被放入全局队列中。

##### Q3.03 调度器的生命周期

![image-20230809230025599](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230809230025599.png)

**特殊的 M0 和 G0**

>M0：M0 是启动程序后的编号为 0 的主线程，这个 M 对应的实例会在全局变量 runtime.m0 中，不需要在 heap 上分配，M0 负责执行初始化操作和启动第一个 G， 在之后 M0 就和其他的 M 一样了。
>
>G0：G0 是每次启动一个 M 都会第一个创建的 gourtine，G0 仅用于负责调度的 G，G0 不指向任何可执行的函数，每个 M 都会有一个自己的 G0。在调度或系统调用时会使用 G0 的栈空间，全局变量的 G0 是 M0 的 G0。

调度器的生命周期几乎占满了一个 Go 程序的一生，runtime.main 的 goroutine 执行之前都是为调度器做准备工作，runtime.main 的 goroutine 运行，才是调度器的真正开始，直到 runtime.main 结束而结束



#### Q4. Go 调度器调度场景过程全解析
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

![image-20230810000042092](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230810000042092.png)

M2 和 P2 会解绑，但 M2 会记住 P2，然后 G8 和 M2 进入系统调用状态。当 G8 和 M2 退出系统调用时，会尝试获取 P2，如果无法获取，则获取空闲的 P，如果依然没有，G8 会被记为可运行状态，并加入到全局队列，M2 因为没有 P 的绑定而变成休眠状态 (长时间休眠等待 GC 回收销毁)。
