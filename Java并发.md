---
title: Java并发编程实战
date: 2021-10-08 12:50:59.663
updated: 2021-10-14 14:30:44.138
url: /archives/jucbasic
categories: 
tags: Java | 并发编程 | 多线程
---

### 在Java中如何实现线程，如何执行线程
---

    第一种方法： **继承Thread类**—>调用*start()*方法，Java虚拟机调用此线程的`run`方法
    
    第二种方法：**实现Runnable接口**+结合Thread类的带参构造—>调用*start()*方法，Java虚拟机调用此线程的`run`方法
    
    第三种方法：**实现Callable接口+Future模式**
    
    第四种方法：**通过线程池 ThreadPoolExecuter**

### 进程与线程的区别
---
- 一个线程只能属于一个进程，而一个进程可以有多个线程，但至少有一个线程。线程依赖于进程而存在
- 进程在执行过程中拥有独立的地址空间，而多个线程共享进程的地址空间。同一进程中的多个线程共享进程的资源。但是每个线程拥有自己的栈段，栈段又叫运行时段，用来存放所有局部变量和临时变量
- 进程是资源分配的最小单位，线程是CPU调度的最小单位
- 进程间切换代价大，线程间切换代价小.进程间不会相互影响；一个进程内某个线程挂掉将导致整个进程挂掉
- 线程共享堆，私有栈
### 死锁的四个必要条件
---
- 1. 互斥条件:任务必须互斥的访问资源
- 2. 请求和保持条件: 任务请求资源时,依旧占有目前其已经持有的资源
- 3. 不剥夺条件: 进程已获得的资源,在末使用完之前,不能强行剥夺
- 4. 循环等待条件: 若干进程之间形成一种头尾相接的循环等待资源关系(n>=2)

### Synchronized和Lock的区别
---

- Synchronized是关键字，内置语言实现，Lock是接口。
- Synchronized在线程发生异常时会自动释放锁，因此不会发生异常死锁。Lock异常时不会自动释放锁，所以需要在finally中实现释放锁。
- Lock是可以中断锁，Synchronized是非中断锁，必须等待线程执行完成释放锁。
- Lock可以使用读锁提高多线程读效率。

### wait()和sleep()的区别 
---
1. **sleep是线程方法，wait是object方法**
2. **sleep：释放cpu资源，不释放锁资源，**如果线程进入sleep的话，释放cpu资源，如果外层包有Synchronize，那么此锁并没有释放掉。**让当前线程休眠进入阻塞状态并释放CPU，但并没有放弃CPU的使用权**
3. **wait，释放cpu资源，也释放锁资源，**一般用于锁机制中 肯定是要释放掉锁的，因为notify并不会立即调起此线程，因此cpu是不会为其分配时间片的，也就是说wait 线程进入等待池，cpu不分时间片给它，锁释放掉。**wait进入等待池，除非被唤醒，否则一直待在里面**
4. .sleep()可以在任何地方使用，wait()只能在同步代码块中使用
---

### sleep()和yield()的区别
1. 线程执行sleep()方法后转入阻塞（BLOCKED）状态，并在“睡眠”一段时间后自动“醒来”，回到就绪（RUNNABLE）状态。而执行yield()方法后，当前线程转入的是就绪（RUNNABLE）状态。

2. 当前线程执行sleep()方法后，其他线程无论优先级高低，都有机会得以运行；而执行yield()方法只会给那些具有相同优先级或更高优先级的线程运行的机会。

3. sleep()方法需要声明抛出InterruptedException，而yield()方法没有声明任何异常。

4. sleep()方法比yield()方法（与操作系统的CPU调度相关）具有更好的可移植性。如果在循环中使用yield()方法，则在Linux上容易导致“死循环”—当前线程在yield()后总是又立即抢占到CPU，导致其他线程得不到执行。
---
### run和start的区别
---
1. **start**():
 Java API中对于该方法的介绍：
    使该线程开始执行；Java 虚拟机调用该线程的 `run` 方法。
    结果是两个线程并发地运行；当前线程（从调用返回给 `start` 方法）和另一个线程（执行其 `run` 方法）。
    多次启动一个线程是非法的。特别是当线程已经结束执行后，不能再重新启动。
    **用start方法来启动线程，真正实现了多线程运行，这时无需等待run方法体中的代码执行完毕而直接继续执行后续的代码。**通过调用Thread类的
    start()方法来**启动一个线程**，这时此线程处于**就绪（可运行）状态，并没有运行**，一旦得到cpu时间片，就开始执行run()方法，这里的run()方法 称为线程体，它包含了要执行的这个线程的内容，Run方法运行结束，此线程随即终止。
2. **run**():
 Java API中对该方法的介绍：
    如果该线程是使用独立的 `Runnable` 运行对象构造的，则调用该 `Runnable` 对象的 `run` 方法；否则，该方法不执行任何操作并返回。
    `  Thread` 的子类应该重写该方法。
    run()方法只是类的一个普通方法而已，如果直接调用Run方法，程序中依然只有主线程这一个线程，其程序执行路径还是只有一条，还是要顺序执行，还是要等待run方法体执行完毕后才可继续执行下面的代码，这样就没有达到写线程的目的。
 3. 总结：

> run()相当于线程的任务处理逻辑的入口方法，它由Java虚拟机在运行相应线程时直接调用，而不是由应用代码进行调用。
而start()的作用是启动相应的线程。启动一个线程实际是请求Java虚拟机运行相应的线程，而这个线程何时能够运行是由线程调度器决定的。start()调用结束并不表示相应线程已经开始运行，这个线程可能稍后运行，也可能永远也不会运行。

**调用start方法方可启动线程，而run方法只是thread类中的一个普通方法调用，还是在主线程里执行**


### Java当中的锁都有哪些
---
公平锁/非公平锁
可重入锁/不可重入锁
自旋锁
乐观锁/悲观锁
偏向锁
轻量级锁/重量级锁

### 线程同步机制
---
1. Object方法 -->  wait() notify()/notifyAll() 必须在同步代码快中才是调用
2. 条件变量Condition方法  --> await()  signal()
3. park / unpark --> LockSupport.park() LockSupport.unpark()  原理类似信号量机制 LockSupport是通过控制变量_counter来对线程阻塞唤醒进行控制的。且_counter的值只有0或者1

### Synchronized的锁升级
![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/20210824222434.png)

---
 a. 轻量级锁和偏向锁的区别

 1. 偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，这样就省去了大量有关锁申请的操作，从而也就提供程序的性能。但是对于锁竞争比较激烈的场合，因为这样场合极有可能每次申请锁的线程都是不相同的，会导致偏向锁失效.
 偏向锁失效后，会先升级为轻量级锁。

 2. 倘若偏向锁失败,虚拟机会尝试使用一种称为轻量级锁的优化手段。此时Mark Word 的结构也变为轻量级锁的结构。轻量级锁能够提升程序性能的依据是“对绝大部分的锁，在整个同步周期内都不存在竞争”。需要了解的是，轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。

 c. Synchronized加锁的类型，类锁和对象锁
  synchronized实现同步的基础:Java中的每一个对象都可以作为锁,具体表现为以下三种形式:
    
    1. 对于普通同步方法,锁是当前实例对象
    2. 对于静态同步方法,锁是当前类的Class对象
    3. 对于同步方法块,锁是synchronized括号里配置的对象
    4. 对于成员方法,锁的是this对象
 d. Synchronized的锁类型
            独占锁，可重入锁

 e. Synchronized经历过怎样的优化<br>

 在JDK1.6中，为了减少获得锁和释放锁带来的性能消耗，引入了偏向锁和轻量级锁，锁的状态变成了四种，如下图所示。锁的状态会随着竞争激烈逐渐升级，但通常情况下，锁的状态只能升级不能降级。这种只能升级不能降级的策略是为了提高获得锁和释放锁的效率。


---
### 线程并发访问一个变量,如何保证安全
1. 将变量定义用synchronized修饰
2. 通过给访问或者修改变量的方法加上锁 ReentrantLock Lock接口的实现类
---
### 如何保证两个线程按照先后顺序依次执行
     wait notify + 修饰的标记(boolean)必须同步
```java
public class Test {
  static  boolean flag = false;
  static Object object = new Object();

  public static void main(String[] args) {
    new Thread(
            () -> {
              synchronized (object) {
                while (!flag) {
                  try {
                    object.wait();
                    System.out.println("a");
                  } catch (InterruptedException e) {
                    e.printStackTrace();
                  }
                }
              }
            })
        .start();
    new Thread(
            () -> {
              synchronized (object) {
                System.out.println("b");
                flag = true;
                object.notifyAll();
              }
            })
        .start();
  }
}

```
    可以使用 LockSupport 类的 park 和 unpark 
    可以使用Lock实现类的lock.newCondition();条件变量await signal
```java
public class Test {
  private static Lock lock = new ReentrantLock();
  private static Condition condition = lock.newCondition();

  public static void main(String[] args) {
    new Thread(
        () -> {
          lock.lock();
          try {
            condition.await();
            System.out.println("a");
            Thread.sleep(1000);
          } catch (InterruptedException e) {
            e.printStackTrace();
          } finally {
            lock.unlock();
          }
        }).start();
    new Thread(
        () -> {
          lock.lock();
          try {
            System.out.println("b");
            condition.signal();
          } catch (Exception e) {
            e.printStackTrace();
          } finally {
            lock.unlock();
          }
        }).start();
  }
}
```

---
### 线程的状态

  Thread.State中枚举除了线程的不同状态
```java
    public enum State {
        NEW,   //   新建
        RUNNABLE, //就绪
        BLOCKED,  //阻塞
        WAITING,  //不见不散
        TIMED_WAITING,//过时不候
        TERMINATED;//终结
    }
```
### Runnable和Callable的区别
 1. 是否有返回值
 2. 是否能抛出异常
 3. 实现方法名称不同,一个是run方法,一个是call方法
---
### synchronized关键字和volatile关键字比较：
1. volatile关键字是线程同步的轻量级实现，所以volatile性能肯定比synchronized关键字要好。但是volatile关键字只能用于变量而synchronized关键字可以修饰方法以及代码块。
2. synchronized关键字在JavaSE1.6之后进行了主要包括为了减少获得锁和释放锁带来的性能消耗而引入的偏向锁和轻量级锁以及其它各种优化之后执行效率有了显著提升，实际开发中使用 synchronized 关键字的场景还是更多一些。
3. 多线程访问volatile关键字不会发生阻塞，而synchronized关键字可能会发生阻塞
4. volatile关键字能保证数据的可见性，但不能保证数据的原子性。synchronized关键字两者都能保证。
5. volatile关键字主要用于解决变量在多个线程之间的可见性，而synchronized关键字解决的是多个线程之间访问资源的同步性。
---
### 线程池的七大参数
![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/XMind%201629813623044.png)

    corePoolSize: 核心线程数
    maximumPoolSize: 能容纳的最大线程数
    keepAliveTime: 救急线程存活时间 (救急线程 = maximumPoolSize - corePoolSize)
    unit 存活时间单位
    workedQueue 阻塞队列(提交但未执行任务的队列)
    threadFactory 创建线程的工厂类 一般是ExectorsdefaultThreadFactory()
    handler,等待队列满后的拒绝策略

###     线程池大小如何设置
CPU 密集型应用，线程池大小设置为 N + 1（N表示CPU数量）

IO 密集型应用，线程池大小设置为 2N

---

### CAS算法
三个操作数:
- 内存值V
- 预估值A
- 更新值B
  
    当且仅当v==A时,v=B,否则将不做任何操作

**可重入: 同一个线程如果首次获得了这把锁,那么因为他是这把锁的拥有者,因此有权利再次获得这把锁,lock和synchronized都是可重入锁**
### 线程池的拒绝策略
1. ThreadPoolExecutor.AbortPolicy 丢弃任务，并抛出 RejectedExecutionException 异常(丢弃并抛出异常)
2. ThreadPoolExecutor.CallerRunsPolicy：该任务被线程池拒绝，由调用 execute方法的线程执行该任务(调用者(发起任务的线程)执行)
3. ThreadPoolExecutor.DiscardOldestPolicy ： 抛弃队列最前面的任务，然后重新尝试执行任务(抛弃等待最久的任务)
4. ThreadPoolExecutor.DiscardPolicy，丢弃任务，不过也不抛出异常(丢弃任务并且什么都不做)
---
### ConcurrentHashMap 知识点
1. ConcurrentHashMap是线程安全的HashMap,与对HashTable进行了改进.对于HashTable,其使用synchronized关键字来保证自身的线程安全.而ConcurrentHashMap(jdk 1.7 -)使用CAS算法+锁分段技术保证了自身的线程安全.每当一个线程占用锁访问一个 Segment 时，不会影响到其他的 Segment。而在jdk1.8+中,由于jdk对synchronized的优化已经很优秀了,而
2. 对HashMap而言,jdk 1.7和jdk 1.8 有很多不同:
    - JDK1.7用的是头插法，而JDK1.8及之后使用的都是尾插法
    - jdk 1.7- 数据结构: 数组+链表
    - jdk 1.8+ 数据结构: 数组+链表 当链表中的元素打到阈值(8) 时,会将链表转化为红黑树,以提高查找效率
3. 为什么在JDK1.8中进行对HashMap优化的时候，把链表转化为红黑树的阈值是8?

由于treenodes的大小大约是常规节点的两倍，因此我们仅在容器包含足够的节点以保证使用时才使用它们，当它们变得太小（由于移除或调整大小）时，它们会被转换回普通的node节点，容器中节点分布在hash桶中的频率遵循泊松分布，桶的长度超过8的概率非常非常小。所以作者应该是根据概率统计而选择了8作为阀值.因为达到8个元素的时候，概率已经很低了。(亿分之六)此时树化，性价比会很高。
既不会因为链表太长(8)导致复杂度加大，也不会因为概率太高导致太多节点树化。

---
### JDK 提供的线程池
---
1. newFixedThreadPool

    - 固定线程数的线程池
    - 阻塞队列使用了LinkedBlockingQueue 链表 无界  
    - 无拒绝策略 不能拒绝任务 不能开辟新线程
    - 适用于任务量已知且相对耗时的任务的场景
2. newCachedThreadPool

    - 核心线程数为0 救急线程可无限创建 且救急线程的空间生存时间为60s
    - 阻塞队列为SynchronousQueue。无容量 没有取 放不入 
    - 适用于任务量大但是任务执行时间短的场景
3. newSingleThreadExecutor

    - 线程池中永远有一个活动线程在处理请求
    - 阻塞队列无界
    - 适用于希望多个任务排队执行的场景
4. newScheduledThreadPool

    - 固定个数的线程池
    - 可以执行延时任务(schedule())
    - 可以执行带有返回值的任务(submit())
### AQS 抽象队列同步器
#### 1.名词解释
---

### synchronized的底层原理
synchronized的底层实现是完全依赖JVM虚拟机的,所以谈synchronized的底层实现，就不得不谈数据在JVM内存的存储：Java对象头，以及Monitor对象监视器。
#### 1.Java对象头
在JVM虚拟机中，对象在内存中的存储布局，可以分为三个区域:
- 对象头(Header)
- 实例数据(Instance Data)
- 对齐填充(Padding)

Java对象头主要包括两部分数据：
1. 类型指针(Class Metadata Address):对象指向它的类元数据的指针,虚拟机通过该指针确定对象是哪个类的实例
2. 标记字段 (Mark Word): 用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等,它是实现轻量级锁和偏向锁的关键.
synchronized使用的锁对象是存储在Java对象头里的标记字段里。


<table>
    <tr>
        <th>锁状态</th> 
        <th>25bit</th> 
        <th>4bit</th> 
        <th>1bit是否是偏向锁</th> 
        <th>2bit 锁标志位</th> 
    </tr>
    <tr>
        <td>无锁状态</td>
        <td>对象HashCode</td>
        <td>对象分代年龄</td>
        <td>0</td>
        <td>01</td>  
    </tr>
    <tr>
        <td>轻量级锁</td>
        <td colspan="3">指向栈中锁记录的指针</td>
        <td>00</td>    
    </tr>
    <tr>
        <td>重量级锁</td>
        <td colspan="3">指向互斥量的指针(指向的是Monitor对象（也称为管程或监视器锁）的起始地址)</td>
        <td>10</td>    
    </tr>
    <tr>
        <td>GC标记</td>
        <td colspan="3">空</td>
        <td>11</td>    
    </tr>
    <tr>
        <td>偏向锁</td>
        <td>线程ID(23bit) + Epoch(2bit)</td>
        <td>对象分代年龄</td>
        <td>1</td>
        <td>01</td>    
    </tr>
</table>

**随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁，但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级**

---

### 8锁案例

```java
public class Lock8Demo {
    private static final Logger LOGGER = LoggerFactory.getLogger(Lock8Demo.class);

     public static void main(String[] args) {
          Phone phone1 = new Phone();
          Phone phone2 = new Phone();

          new Thread(()->{
               phone1.sendEmail();
          },"a").start();
          // 暂停 保证a线程先启动
          try {
               TimeUnit.SECONDS.sleep(1);
          } catch (InterruptedException e) {
               LOGGER.error("异常", e);
          }
          new Thread(()->{
               phone2.sendSMS();
//               phone.hello();
          },"b").start();
     }


}
// 资源类
class Phone{
     private static final Logger LOGGER = LoggerFactory.getLogger(Phone.class);

     public static synchronized void sendEmail(){
          try {
               TimeUnit.SECONDS.sleep(3);
          } catch (InterruptedException e) {
               LOGGER.error("异常", e);
          }
          LOGGER.info("======= sendEmail");
     }
     public  synchronized void sendSMS(){
          LOGGER.info("======= sendSMS");
     }

     public void hello(){
          LOGGER.info("======= hello");
     }
}
```

> 1. synchronized修饰普通成员方法,锁的是当前对象this,被锁定后其他线程都不能进入到当前对象的其他synchronized方法(对象锁)
> 2. 普通成员方法和synchronized修饰的成员方法并不构成竞态条件
> 3. synchronized修饰静态成员方法,锁的是当前类的Class对象(类锁)
> 4. 对于同步方法块,锁的是synchronized括号中的对象
> 5. 静态同步方法和普通同步方法因为持有的锁对象不同,不会构成竞态条件

#### 创建线程池

```java
new ThreadPoolExecutor(
             CORE+1,
             CORE+2,
             60,
             TimeUnit.SECONDS,
              new ArrayBlockingQueue<>(50,true),
             new ThreadFactoryBuilder().setNameFormat("computationally-Intensive-Pool").build(),
             new ThreadPoolExecutor.AbortPolicy()
     );
```

### JMM(Java内存模型)的三大特性:

- > 可见性 当线程修改共享变量后,其他线程是否能立即接收到最新的数据变更

- > 原子性 

- > 有序性 只要程序的最终结果与它顺序化执行结果相等,那么为了提升性能,编译器和处理器通常会对指令序列进行重新排序,导致指令的执行顺序与代码顺序不一致(指令重排)

### JMM happens-before

在JMM 中,如果一个操作执行的结果需要对另一个操作可见性 或者代码重排序 那么这两个操作之间必须存在happens-before先行发生原则

>1. 如果一个操作happens-before另一个操作,那么第一个操作的执行结果将对第二个操作课件,而且第一个操作的执行顺序排在第二个操作之前(顺序)
>2. 两个操作之间存在happens-before关系,并不意味着一定要按照happens-before原则制定的顺序来执行 如果重排序之后的执行结果与按照happens-before关系来执行的结果一致,那么这种重排序并不非法(重排)

### volatile与内存屏障

- 当写一个volatile变量时,JMM会把该线程对应的本地内存中的共享变量值**立即刷新回主内存**中
- 当读一个volatile变量时, JMM会把该线程对应的本地内存设置为无效,重新回到主内存中读取最新共享变量

> 内存屏障是一类同步屏障指令,是CPU或编译器在对内存随机访问操作中的一个同步点,使得此点之前的所有读写操作都执行后才可以开始执行此点之后的操作,避免了代码重排序
>
> 内存屏障就是一种JVM指令,Java内存模型的重排规则会**要求Java编译器在生成JVM指令时插入特定的内存屏障指令**,通过这些内存屏障指令,volatile实现Java内存模型中的可见性和有序性(禁重排) **但volatile无法保证原子性**
>
> *内存屏障之前*的所有**写操作**都要**回写到主内存**
>
> *内存屏障之后*的所有**读操作**都能获得内存屏障之前的所有写操作的最新结果



在每一个volatile写操作前面JVM都插入一个StoreStore屏障(禁止之前的普通写与下面的volatile写重排序)

在每一个volatile写操作后面JVM都插入一个StoreLoad屏障(禁止之前的volatile写与下面可能有的volatile读写重排序)

### ThreadLocal最佳实践

>1. ThreadLocal.withInitial(()->初始化值) **使用之前先初始化,避免空指针异常**
>2. 建议把ThreadLocal修饰为static
>3. **用完记得手动remove()  **    避免线程池不及时清除导致脏数据复用

---

### 对象内存布局

对象头+实例数据+对齐填充(保证8字节的倍数)

对象头 = 对象标记(Mark Word) + 类元信息(类型指针)

对象标记保存了哈希码/GC标记/GC次数/同步锁标记/偏向锁持有者等信息

### synchronized锁的种类以及锁升级流程

> synchronized用的锁是存在Java对象头里的Mark Word中
>
> 锁升级功能主要依赖Mark Word中锁标志位和释放偏向锁标志位

- 偏向锁 *当一段同步代码块一直被同一个线程多次访问,由于只有一个线程那么该线程在后续访问时便会自动获得锁*: MarkWord存储的是偏向的线程ID;(单线程竞争)
- 轻量锁: MarkWord存储的是指向**线程栈**中Lock Record的指针;
- 重量锁: MarkWord存储的是指向**堆**中Monitor对象的指针

> 偏向锁和对象一致性哈希 不共存
>
> - 本应是偏向锁,调用对象hashCode()方法,就会升级为轻量级锁
> - 已经是偏向锁,调用对象hashCode()方法,就会升级为重量级锁

```java
package com.coder4j.myplayer.juc.completablefuture;

import org.junit.jupiter.api.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @author gangchengcn@gmail.com
 * @Description TODO
 * @date 2023/3/24-16:34
 */

public class CompletableFutureTest {

    private static final Logger LOG = LoggerFactory.getLogger(CompletableFutureTest.class);

    private static final ExecutorService executor = Executors.newFixedThreadPool(5);

    /**
     * 测试零依赖CompletableFuture的创建和使用
     */
    @Test
    void zeroDependencyTest() {
        LOG.info("零依赖：CompletableFuture的创建");
        // 1、使用runAsync或supplyAsync发起异步调用
        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            try {
                int i = 10 / 0;
                Thread.sleep(300);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            LOG.info("result1");
            return "result1";
        }, executor);
        LOG.info("cf1 --> {}", cf1);
        LOG.info(cf1.join());
        LOG.info("cf1 --> {}", cf1);
        LOG.info("执行结束");
    }

    /**
     * 测试单依赖CompletableFuture的创建和使用
     * CompletableFuture的单依赖可以通过thenApply、thenAccept、thenCompose等方法来实现
     *
     * @throws ExecutionException
     * @throws InterruptedException
     */
    @Test
    void monadicDependenceTest() throws ExecutionException, InterruptedException {
        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            LOG.info("result1");
            return "result1";
        }, executor);
        CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
            LOG.info("result2");
            return "result2";
        }, executor);

        CompletableFuture<String> cf3 = cf1.thenApply(result1 -> {
            // result1为CF1的结果
            //......
            LOG.info("result3");
            return "result3";
        });
        CompletableFuture<String> cf5 = cf2.thenApply(result2 -> {
            // result2为CF2的结果
            //......
            LOG.info("result5");
            return "result5";
        });
        cf3.get();
        cf5.get();
        LOG.info("执行结束");
    }


    /**
     * 测试多依赖CompletableFuture的创建和使用
     *         多元依赖可以通过allOf或anyOf方法来实现，区别是当需要多个依赖全部完成时使用allOf，当多个依赖中的任意一个完成即可时使用anyOf
     * @throws ExecutionException
     * @throws InterruptedException
     */
    @Test
    void multipleDependenceTest() throws ExecutionException, InterruptedException {
        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            LOG.info("result1");
            return "result1";
        }, executor);
        CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
            LOG.info("result2");
            return "result2";
        }, executor);
        CompletableFuture<String> cf3 = cf1.thenApply(result1 -> {
            // result1为CF1的结果
            //......
            LOG.info("result3");
            return "result3";
        });
        CompletableFuture<String> cf5 = cf2.thenApply(result2 -> {
            // result2为CF2的结果
            LOG.info("result5");
            return "result5";
        });
        CompletableFuture<Void> cf6 = CompletableFuture.allOf(cf1, cf2, cf5);
        cf6.get();
        CompletableFuture<String> cf4 = cf1.thenCombine(cf2, (result1, result2) -> {
            // result1和result2分别为cf1和cf2的结果
            LOG.info("result4");
            LOG.info(result1 + "...." + result2);
            return "result4";
        });
        CompletableFuture<String> result = cf6.thenApply(v -> {
            // 这里的join并不会阻塞，因为传给thenApply的函数是在CF3、CF4、CF5全部完成时，才会执行 。
            String result3 = cf3.join();
            String result4 = cf4.join();
            String result5 = cf5.join();
            LOG.info(result3 + "-" + result4 + "-" + result5);
            LOG.info("end");
            return "end";
        });

        result.get();
    }

}
```

