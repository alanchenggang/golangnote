# JVM	

## 内存池划分
---

![image-20230822203935755](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230822203935755.png)

###  新生代（Eden Space）

Eden Space，也叫伊甸区，是内存中的一个区域，用来分配新创建的对象。通常会有多个线程同时创建多个对象，所以 Eden 区被划分为多个 **线程本地分配缓冲区**（Thread Local Allocation Buffer，简称 TLAB）。通过这种缓冲区划分，大部分对象直接由 JVM 在对应线程的 TLAB 中分配，避免与其他线程的同步操作。

如果 TLAB 中没有足够的内存空间，就会在共享 Eden 区（shared Eden space）之中分配。如果共享 Eden 区也没有足够的空间，就会触发一次 年轻代 GC 来释放内存空间。如果 GC 之后 Eden 区依然没有足够的空闲内存区域，则对象就会被分配到老年代空间（Old Generation）。

当 Eden 区进行垃圾收集时，GC 将所有从 root 可达的对象过一遍，并标记为存活对象。

我们曾指出，对象间可能会有跨代的引用，所以需要一种方法来标记从其他分代中指向 Eden 的所有引用。这样做又会遭遇各个分代之间一遍又一遍的引用。JVM 在实现时采用了一些绝招：卡片标记（card-marking）。从本质上讲，JVM 只需要记住 Eden 区中“脏”对象的粗略位置，可能有老年代的对象引用指向这部分区间

标记阶段完成后，Eden 区中所有存活的对象都会被复制到存活区（Survivor spaces）里面。整个 Eden 区就可以被认为是空的，然后就能用来分配新对象。这种方法称为“**标记—复制**”（Mark and Copy）：存活的对象被标记，然后复制到一个存活区（注意，是复制，而不是移动）。

### 存活区（Survivor Spaces）

Eden 区的旁边是两个**存活区**（Survivor Spaces），称为 from 空间和 to 空间。需要着重强调的的是，任意时刻总有一个存活区是空的（empty）。

空的那个存活区用于在下一次年轻代 GC 时存放收集的对象。年轻代中所有的存活对象（包括 Eden 区和非空的那个“from”存活区）都会被复制到 ”to“ 存活区。GC 过程完成后，“to”区有对象，而“from”区里没有对象。两者的角色进行正好切换，from 变成 to，to 变成 from。

![image-20230822204254440](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230822204254440.png)

存活的对象会在两个存活区之间复制多次，直到某些对象的存活 时间达到一定的阀值。分代理论假设，存活超过一定时间的对象很可能会继续存活更长时间。

这类“年老”的对象因此被**提升**（promoted）到老年代。提升的时候，存活区的对象不再是复制到另一个存活区，而是迁移到老年代，并在老年代一直驻留，直到变为不可达对象。

为了确定一个对象是否“足够老”，可以被提升（Promotion）到老年代，GC 模块跟踪记录每个存活区对象存活的次数。每次分代 GC 完成后，存活对象的年龄就会增长。当年龄超过**提升阈值**（tenuring threshold），就会被提升到老年代区域。

具体的提升阈值由 JVM 动态调整，但也可以用参数 `-XX:+MaxTenuringThreshold` 来指定上限。如果设置 `-XX:+MaxTenuringThreshold=0` ，则 GC 时存活对象不在存活区之间复制，直接提升到老年代。现代 JVM 中这个阈值默认设置为 **15** 个 GC 周期。这也是 HotSpot JVM 中允许的最大值。

如果存活区空间不够存放年轻代中的存活对象，提升（Promotion）也可能更早地进行。

### 老年代（Old Gen）

老年代 GC 发生的频率比年轻代小很多。同时，因为预期老年代中的对象大部分是存活的，所以不再使用标记和复制（Mark and Copy）算法。而是采用移动对象的方式来实现最小化内存碎片。老年代空间的清理算法通常是建立在不同的基础上的。原则上，会执行以下这些步骤：

- 通过标志位（marked bit），标记所有通过 GC roots 可达的对象；
- 删除所有不可达对象；
- 整理老年代空间中的内容，方法是将所有的存活对象复制，从老年代空间开始的地方依次存放。

通过上面的描述可知，老年代 GC 必须明确地进行整理，以避免内存碎片过多。

### 永久代（Perm Gen）

在 Java 8 之前有一个特殊的空间，称为“永久代”（Permanent Generation）。这是存储元数据（metadata）的地方，比如 class 信息等。此外，这个区域中也保存有其他的数据和信息，包括内部化的字符串（internalized strings）等等。

### 元数据区（Metaspace）

Java 8 直接删除了永久代（Permanent Generation），改用 Metaspace。从此以后，Java 中很多杂七杂八的东西都放置到普通的堆内存里。

当然，像类定义（class definitions）之类的信息会被加载到 Metaspace 中。元数据区位于本地内存（native memory），不再影响到普通的 Java 对象。默认情况下，Metaspace 的大小只受限于 Java 进程可用的本地内存。这样程序就不再因为多加载了几个类/JAR 包就导致 `java.lang.OutOfMemoryError: Permgen space.`。注意，这种不受限制的空间也不是没有代价的 —— 如果 Metaspace 失控，则可能会导致严重影响程序性能的内存交换（swapping），或者导致本地内存分配失败

如果需要避免这种最坏情况，那么可以通过下面这样的方式来限制 Metaspace 的大小，如 256 MB：

```diff
-XX:MaxMetaspaceSize=256m
```



---

![72433648.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/bf6df0c0-32ee-11ea-b0e0-6da2f5afc39e)

---
## JVM启动流程探究
---
### 1. 配置JVM装载环境

### 2. 解析虚拟机参数

### 3. 设置线程栈大小

### 4. 执行JavaMain方法

## 垃圾回收算法
---
### 1. 标记阶段

#### 引用计数算法

![image-20230301213821280](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230301213821280.png)

#### 可达性分析算法(GCROOTS)

![image-20230301214554132](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230301214554132.png)

### 2. 清除阶段

#### 标记-清除算法(Mark-Sweep)

![image-20230301215705758](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230301215705758.png)

![image-20230301220244414](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230301220244414.png)

#### 复制算法

![image-20230301220856982](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230301220856982.png)

![image-20230301220932767](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230301220932767.png)

![image-20230301221332481](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230301221332481.png)

#### 标记-整理算法

![image-20230301222950868](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230301222950868.png)

![image-20230301222914374](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230301222914374.png)



## 运行时数据区
1. 程序计数器 PC 指向下一条指令的地址  线程私有 没有OOM异常
2. 虚拟机栈  基本单位--> 栈帧 一个线程对应一个栈帧  会发生OOM
3. 本地方法栈
4. 堆 
整个Java应用程序共享的区域.职责就是存储和管理对象和数组 GC机制主要发生作用的内存区域
5. 方法区
整个Java应用程序共享的区域.存储所有的类信息/常量/静态变量/动态编译缓存等数据 方法区= 类信息表+运行时常量池. 运行时常量池存放各个类的类常量池信息
---
## Java中的类加载器
Java 中的类加载器大致可以分成两类，一类是系统提供的，另外一类则是由Java 应用开发人员编写的。系统提供的类加载器主要有下面三个：



### 引导类加载器（bootstrap class loader）：

它用来加载 Java 的核心库，是用原生代码来实现的，并不继承自 java.lang.ClassLoader。主要负责jdk_home/lib目录下的核心api 或 -Xbootclasspath 选项指定的jar包装入工作（其中的jdk_home是指配置jdk环境变量是java_home的配置路径，一般是jdk/jre所在目录）。

### 扩展类加载器（extensions class loader）：

它用来加载 Java 的扩展库。Java虚拟机的实现会提供一个扩展库目录，扩展类加载器在此目录里面查找并加载 Java 类，主要负责jdk_home/lib/ext目录下的jar包或 -Djava.ext.dirs 指定目录下的jar包装入工作。

### 系统类加载器（system class loader）：

它根据 Java 应用的类路径（CLASSPATH）来加载 Java 类。一般来说，Java 应用的类都是由它来完成加载的。可以通过 ClassLoader.getSystemClassLoader()来获取它。主要负责CLASSPATH/-Djava.class.path所指的目录下的类与jar包装入工作.

---
## 谈一谈Java中的四大引用
---
1. 强引用:
    - new出来的对象.在代码中最常见
    - 无论何种情况,只要强引用关系还存在,GC就永远不会回收掉被引用的对象
2. 软引用
    - 描述一些还有用,但非必须的对象
    - 内存不够才回收,回收软引用对象后仍然内存不够,才报出内存溢出异常
    - 通过jdk提供的SoftReference类来实现软引用
3. 弱引用
    - 描述那些比软引用更弱一点的对象
    - 只能生存到下一次GC发生为止,不论剩余内存大小都回收
    - 通过jdk提供的WeakReference类来实现软引用
4. 虚引用
    - 为一个对象设置虚引用的唯一目的就是为了能在这个对象被收集器回收时收到一个系统通知
    - 通过jdk提供的PhantomReference类来实现软引用
## JVM中锁的关键字
---

## JVM内存模型
---
<img src="https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/1628307-20191231224756735-422346379.png">

<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">JVM内存模型图解</div>
</center>

> Java 堆内存(Heap)、方法区(Non-Heap)、JVM栈(JVM Stack)、本地方法栈（Native Method Stacks）、程序计数器（Program Counter Register）

### Java 堆内存(Heap)
- 所有线程共享
- 存放对象实例
- 垃圾收集器管理的主要区域(GC堆)
- 会发生OOM异常
### 方法区(Non-Heap)
- 所有线程共享
- 用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据
- 逐步转移到Java Heap 在Java8之后被元空间(Metaspace)所取代
### 程序计数器（Program Counter Register）
- 指向下一条要执行的字节码指令的位置
- 线程私有
### JVM栈(JVM Stack)
- 线程私有,生命周期与线程相同
- 描述的是Java方法执行的内存模型,每个方法执行的同时创建**栈帧**,方法被调用到执行完成对应着栈帧从入栈到出栈的过程
- 如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常 如果虚拟机栈可以动态扩展，当扩展时无法申请到足够的内存时会抛出OutOfMemoryError异常。
#### 栈帧(Stack Frame）
- 局部变量表 局部变量表是一组变量值的存储空间，用于存放方法参数和方法内部定义的局部变量
- 操作数栈 借助栈完成计算等操作
- 动态连接 每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接
> class文件的常量池中存有大量的符号引用，字节码中的方法调用指令就以常量池里指向方法的符号引用作为参数，这些符号引用会在类加载阶段或者第一次使用的时候转化为直接引用，这个转化称为静态解析。另外一部分将在每一次运行期间转化为直接引用，这部分就称为动态连接
- 返回地址 (正常调用完成/异常调用完成)
#### 本地方法栈（Native Method Stacks)
- 虚拟机执行使用到的Native方法服务
- 会抛出StackOverflowError和OutOfMemoryError异常
- 与其他语音交互

<img src="https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/1628307-20191231224451199-929179305.png">

<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">JVM线程共享区域图解</div>
</center>

## 类的生命周期和类加载
---
## 介绍一下双亲委派机制


## 如何打破双亲委派机制(Tomcat)

---
## 介绍垃圾回收算法
标记清除算法
标记整理算法

---
## 介绍一下各种垃圾回收器,并说明他们有何特点
---
##  介绍一下STW机制

在标记整理算法中,移动存活对象并更新所有引用这些对象的地方将会是一种极为负重的操作,而且这种对象移动操作必须全程暂停用户应用程序才能进行;

根节点枚举必须保证始终还是必须在一个能保障一致性的快照中才能得以运行,否则不能保分析结果的准确性,也会造成用户线程面临STW的困扰

---
## 介绍一下GCROOTS的构成


---

- 在虚拟机栈(栈帧中的本地变量表)中引用的对象(当前方法的参数/局部变量/临时变量等)

- 在方法区中类静态属性引用的对象(Java类的引用类型静态变量)

- 在方法区中常量引用的对象(字符串常量池里的引用)

- 在本地方法栈中JNI引用的对象

- Java虚拟机内部的引用(基本数据类型对应的Class对象/一些常驻异常类/系统类加载器)

- 所有被同步锁(synchronized关键字)持有的对象

- 反映Java虚拟机内部情况的JMXBean/JVMTI中注册的回调/本地代码缓存等

  ![image-20230301215204010](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230301215204010.png)

## 介绍下三色标记算法

遍历对象图过程中遇到的对象,按照“是否访问过”这个条件标记成以下三种颜色:

- 白色: 表示对象尚未被垃圾收集器访问过
- 黑色:表示对象已经被垃圾收集器访问过,且这个对象的引用都已经扫描过,他是安全存活的对象(**黑色对象不可能直接指向某个白色对象**)
- 灰色:表示对象已经被垃圾收集器访问,但是这个对象的引用还没有被收集器扫描完



## 什么是安全点

![image-20230303112147206](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230303112147206.png)

![image-20230303112516528](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230303112516528.png)

#### 安全区域

![image-20230303112551940](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230303112551940.png)

##  7款经典垃圾收集器

### Serial收集器(串行回收)

- 单线程工作
- 基于标记-复制算法实现
- GC会发生STW
- 简单而且高效,内存占用小,没有线程交互开销
- 能与CMS收集器配合工作(9-)

### ParNew收集器

- Serial收集器的多线程**并行**版本
- 基于标记-复制算法实现
- 能与CMS收集器配合工作(9/9+后成为唯一搭配)

### Parallel Scavenge收集器

- 基于标记-复制算法实现
- 能够并行收集的多线程收集器
- 自适应调节  目标:达到一个可控制吞吐量(吞吐量优先收集器)
- -XX:MaxGCPauseMillis 最大垃圾收集停顿时间
  -XX:GCTimeRatio 设置吞吐量大小

### Serial Old 收集器

- Serial收集器的老年代版本
- 单线程收集器
- 基于标记-整理算法实现

### Parallel Old 收集器

- Parallel Scavenge收集器的老年代版本

- 多线程并行收集

- 基于标记-整理算法实现

  

### CMS(Concurrent Mark Sweep)收集器

- 目标为获取最短回收停顿时间的收集器(并发低停顿收集器)

- 基于标记-清除算法实现

- >GC四阶段:
  >
  >​    初始标记(STW)  并发标记 重新标记(STW) 并发清除

- 缺点
  - CMS收集器对处理器资源非常敏感(占用过多处理器资源)
  - 由于垃圾收集阶段用户线程需要持续运行,需要预留足够的老年代空间(JDK5: 老年代使用68%触发 JDK6:老年代使用92%触发 参数-XX:CMSInitiatingOccupancyFeaction设置CMS触发百分比)提供用户线程使用,无法处理浮动垃圾,有可能出现并发失败而导致一次完全的STW的Full GC发生
  - 空间碎片产生,对大对象不友好

### G1收集器

- 面向局部收集的设计思想

- 基于Region的内存布局形式

- 9+默认收集器

![image-20230303213626241](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230303213626241.png)

![image-20230303213919451](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230303213919451.png)

#### 垃圾回收器与垃圾分代之间的关系

![image-20230303214107481](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230303214107481.png)

#### 组合方式

![image-20230303214456981](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230303214456981.png)

![image-20230303214741251](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230303214741251.png)




## Java中常见的内存泄漏

![image-20230303110547836](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230303110547836.png)

## 四大引用

![image-20230303114200470](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com//md/imgimage-20230303114200470.png)
