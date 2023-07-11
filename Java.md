---
title: Java基础
date: 2021-10-08 12:59:54.604
updated: 2021-10-13 15:58:37.201
url: /archives/javabasic
categories: 
tags: 基础 | Java
---

# 集合
---
### 集合的层次结构
![集合的层次结构](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/%E9%9B%86%E5%90%88%E7%9A%84%E5%B1%82%E6%AC%A1%E7%BB%93%E6%9E%84.png)
---
<img src="https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/2970531_1557843217158_F6C79F5BE8BA80BAA8CB04BB40951DBB" alt="img" style="zoom:200%;" />

Java 集合类库采用 “持有对象”（holding objects）的思想，并将其分为两个不同的
概念，表示为类库的基本接口：
1. 集合（Collection）：一个独立元素的序列，这些元素都服从一条或多条规则。List
必须以插入的顺序保存元素，Set 不能包含重复元素，Queue 按照排队规则来确
定对象产生的顺序（通常与它们被插入的顺序相同）。
第十二章集合 6
2. 映射（Map）：一组成对的 “键值对” 对象，允许使用键来查找值。ArrayList 使用
数字来查找对象，因此在某种意义上讲，它是将数字和对象关联在一起。map 允
许我们使用一个对象来查找另一个对象，它也被称作关联数组（associative array），
因为它将对象和其它对象关联在一起；或者称作字典（dictionary），因为可以使
用一个键对象来查找值对象，就像在字典中使用单词查找定义一样。Map 是强大
的编程工具。
---
### ArrayList和LinkedList区别

| ArrayList<E>               | LinkedList<E>                  | Vector<E>              |
| -------------------------- | ------------------------------ | ---------------------- |
| 底层是有**动态数组**维护的 | 底层是由**双向链表**维护       | 底层是有动态数组维护的 |
| **查询最快**               | **添加/删除效率高   查询最慢** | 都慢                   |
| 线程不安全                 | 线程不安全                     | 线程安全               |

    Arraylist默认数组大小是10，扩容后的大小是扩容前的1.5倍，最大值小于Integer 的最大值减8
    如果新创建的集合有带初始值，默认就是传入的大小，也就不会扩容
增强for循环只能用于遍历集合，而不能对集合元素进行操作。否则将出现并发修改异常

    对集合遍历且需要修改元素时，应使用普通for循环/迭代器
    
    迭代器在迭代过程中要注意的事项：

1.迭代器在迭代过程中不允许使用集合对象改变集合的元素个数。否则会报出：ConcurrentModificationException，即并发修改异常。      
2.在迭代过程中如果需要改变集合中的元素个数，只能使用迭代器的方法去改变。如add()方法和remove()方法。
---
### Map<K,V>集合的对比

| HashMap<K,V>          | LinkedHashMap<K,V>                 | TreeMap<K,V>                      | HashTable<K,V>        | ConcurrentHashMap<K,V>              |
| --------------------- | ---------------------------------- | --------------------------------- | --------------------- | ----------------------------------- |
| 元素无序(根据key)     | 元素有序(插入顺序遍历顺序保持一致) | 元素有序(按照key值的自然顺序排列) |                       |                                     |
| 底层由Hash表维护      | 底层由Hash表和链表维护             | 底层由红黑树维护                  | 底层由Hash表维护      | 底层由CAS(比较并交换)乐观锁机制维护 |
| 线程不安全            | 线程不安全                         | 线程不安全                        | 线程安全(悲观锁)      | 线程安全(乐观锁)                    |
| key/value都可以为null | key/value都可以为null              | key不能为null,value可以为null     | key/value都不能为null | key/value都不能为null               |

> 线程安全的map:**HashTable,SynchronizedMap,ConcurrentHashMap**

---
### HashMap底层数据结构，是否线程安全？线程安全要怎么做？
HashMap底层是由Hash表维护.并且按照链地址法解决冲突

在JDK1.6，JDK1.7中，HashMap采用位桶+链表实现，即使用链表处理冲突，同一hash值的链表都存储在一个链表里。但是当位于一个桶中的元素较多，即hash值相等的元素较多时，通过key值依次查找的效率较低。

而JDK1.8中，HashMap采用位桶+链表+红黑树实现，当链表长度超过阈值（8）时，将链表转换为红黑树，这样大大减少了查找时间


HashMap非线程安全

HashMap线程安全的做法:
- 1.使用HashTable:
hashtable底层是数组+链表的形式，其中的get、put方法等都是用synchronized修饰的，因此，hashtable是线程安全的，但是执行效率比较低，一般不推荐使用。
- 2.使用Collections类的静态方法synchronizedMap获得线程安全的HashMap。synchronizedMap的线程安全和hashtable一样，都是使用sunchronized方法，将方法上锁。 
- 3.使用java.util.concurrent.concurentHashMap
使用currentHashMap的锁分段技术可以有效的提高并发访问率

HashMap为何每次扩容是原来的2倍:

    HashMap计算添加元素的位置时，使用的位运算，这是特别高效的运算
    HashMap的初始容量是2的n次幂，扩容也是2倍的形式进行扩容，是因为容量是2的n次幂，可以使得添加的元素均匀分布在HashMap中的数组上，减少hash碰撞，避免形成链表的结构，使得查询效率降低
---
### ArrayList去重怎么实现（不用set）
对list集合中的重复值进行处理，大部分是采用两种方法，

　　一种是用遍历list集合判断后赋给另一个list集合，
```java
　　if (!newList.contains(cd))
```
　　另一种是用赋给set集合再返回给list集合。 

---
### List和Map的区别

- List：是存储单列数据的集合，存储的数据是有序并且是可以重复的，实现的是Collection接口

- Map：存储双列数据的集合，通过键值对存储数据，存储 的数据是无序的，Key值不能重复，value值可以重复，实现的是Map接口
---
### 对Map进行遍历
---
1. 通过 keySet() 或 values() 方法遍历
```java
     public void testKeySet() {
        for (Integer key : map.keySet()) {
          System.out.println(key);
        }
    }
    /** values 获取 value */
    public void testValues() {
        for (Integer value : map.values()) {
            System.out.println(value);
        }
    }
```

2. 通过 keySet 的 get(key) 获取值
```java
      /**
   * 
   * @param map  要遍历的map
   */
  private static void setSeeAll(Map<Integer, String> map) {
    Set<Integer> keySet = map.keySet();
    for (Integer key :keySet ) {
      System.out.println("Key:"+key+";Value:"+map.get(key));
    }
  }
```
3. 通过 entrySet 遍历
```java
  /**
   * 通过 entrySet 遍历 通过对 Map 的 entrySet 进行遍历, 也可以同时拿到 key 和 value.
   *
   * <p>该方法是最常用的遍历 Map 的方法, 大多情况下, 其性能要优于第2种.
   *
   * @param map
   */
  private static void entrySetSeeAll(Map<Integer, String> map) {
    for (Map.Entry<Integer, String> entry : map.entrySet()) {
      System.out.println(entry.getKey() + ":" + entry.getValue());
    }
  }
```
4. 通过迭代器 Iterator 遍历
```java
  /**
   * 通过迭代器 Iterator 遍历 上面的几种遍历方法都用了增强型 for 循环 —— foreach, 这是 JDK 5 开始才有的特性.
   *
   * <p>foreach 的操作虽然看起来很简洁, 但有一个劣势: 遍历 Map 时, 如果改变其大小, 就会抛出并发修改异常. 但如果在遍历时只需要删除 Map 中的元素, 那就可以用
   * Iterator 的 remove() 方法删除元素:
   *
   * @param map
   */
  private static void iterableSeeAll(Map<Integer, String> map) {
    Iterator<Map.Entry<Integer, String>> entryIterator = map.entrySet().iterator();
    while (entryIterator.hasNext()){
      Map.Entry<Integer, String> entry = entryIterator.next();
      System.out.println(entry.getKey()+":"+entry.getValue());
    }
  }
```
迭代器通常被称为轻量级对象（lightweight object）：创建它的代价小。
因此，经常可以看到一些对迭代器有些奇怪的约束Java 的 Iterator 只能单向
移动。迭代器的优点:能够将遍历序列的操作与该序列的底层结构分离.
这个 Iterator 只能用来：
1. 使用 iterator() 方法要求集合返回一个 Iterator。Iterator 将准备好返回序列
中的第一个元素。
2. 使用 next() 方法获得序列中的下一个元素。
3. 使用 hasNext() 方法检查序列中是否还有元素。
4. 使用 remove() 方法将迭代器最近返回的那个元素删除
#### ListIterator
---
ListIterator 是一个更强大的 Iterator 子类型，它只能由各种 List 类生成。Iter-
ator 只能向前移动，而 ListIterator 可以双向移动。它可以生成迭代器在列表中指向
位置的后一个和前一个元素的索引，并且可以使用 set() 方法替换它访问过的最近一
个元素。可以通过调用 listIterator() 方法来生成指向 List 开头处的 ListIterator ，
还可以通过调用 listIterator(n) 创建一个一开始就指向列表索引号为 n 的元素处的
ListIterator 。
---
5. 通过 Lambda 表达式遍历
```java
  /**
   * JDK 8 提供了 Lambda 表达式支持, 其语法看起来更简洁, 可以同时拿到 key 和 value.
   *
   * <p>不过, 经过简单的测试, Lambda 表达式遍历 Map 的速度要低于 entrySet 遍历的方式, 所以更推荐用 entrySet 去遍历 Map.
   *
   * @param map
   */
  private static void lambdaSeeAll(Map<Integer, String> map) {
    map.forEach(
        (key, value) -> {
          System.out.println(key+":"+value);
        });
  }
```
综上所述:

    (1) 如果只获取 key 或者 value, 推荐使用 keySet() 或 values() 方法;
    (2) 如果需要同时获取 key 和value, 推荐使用 entrySet;
    (3) 如果需要在遍历过程中删除元素, 推荐使用 Iterator;
    (4) 如果需要在遍历过程中添加元素, 可以新建一个临时 Map 存放新增的元素, 遍历结束后, 再把临时 Map 添加到原 Map 中.

## 集合的各种操作
从大到小
```java
        Comparator<Object> reverseOrder = Collections.reverseOrder();
        list.sort(reverseOrder);
```
---
# 异常

### Exception和Error的区别

   >Error类和Exception类的父类都是throwable类，他们的区别是： 
   >
   >Error类一般是指与***虚拟机相关的问题***，如**系统崩溃，虚拟机错误，内存空间不足，方法调用栈溢**等。对于这类错误的导致的应用程序中断，仅靠程序本身无法恢复和和预防，遇到这样的错误，**建议让程序终止**。 
   >
   >Exception类表示**程序可以处理的异常**，可以**捕获**且可能*恢复*。遇到这类异常，应该**尽可能处理异常，使程序恢复运行，而不应该随意终止异常**。 
   >
   >Exception类又分为**运行时异常（Runtime Exception）**和**受检查的异常(Checked Exception  )**，运行时异常;ArithmaticException,IllegalArgumentException，**编译能通过**，但是**一运行就终止**了，**程序不会处理运行时异常**，出现这类异常，程序会终止。而受检查的异常，要么用***try。。。catch捕获，要么用throws字句声明抛出**，交给它的父类处理，否则**编译不会通过**.
---
### 常用的异常

- Java中常见运行时异常

|           异常类型            |                         说明                          |
| :---------------------------: | :---------------------------------------------------: |
|      ArithmeticException      |              算术错误异常，如以零做除数               |
| ArraylndexOutOfBoundException |                     数组索引越界                      |
|      ArrayStoreException      |              向类型不兼容的数组元素赋值               |
|      ClassCastException       |                     类型转换异常                      |
|   IllegalArgumentException    |                 使用非法实参调用方法                  |
|     lIIegalStateException     |            环境或应用程序处于不正确的状态             |
|  lIIegalThreadStateException  |           被请求的操作与当前线程状态不兼容            |
|   IndexOutOfBoundsException   |                  某种类型的索引越界                   |
|     NullPointerException      |          尝试访问 null 对象成员，空指针异常           |
|  NegativeArraySizeException   |                再负数范围内创建的数组                 |
|     NumberFormatException     | 数字转化格式异常，比如字符串到 float 型数字的转换无效 |
|    TypeNotPresentException    |                      类型未找到                       |

- Java常见非运行时异常

  |           异常类型           |            说明            |
  | :--------------------------: | :------------------------: |
  |    ClassNotFoundException    |         没有找到类         |
  |    IllegalAccessException    |        访问类被拒绝        |
  |    InstantiationException    | 试图创建抽象类或接口的对象 |
  |     InterruptedException     |    线程被另一个线程中断    |
  |     NoSuchFieldException     |       请求的域不存在       |
  |    NoSuchMethodException     |      请求的方法不存在      |
  | ReflectiveOperationException |   与反射有关的异常的超类   |
---
### 处理异常的执行过程

>1、finally块一定会执行，无论是否try…catch。
>
>2、finally前有return，会先执行return语句，并保存下来，再执行finally块，最后return。
>
>3、finally前有return、finally块中也有return，先执行前面的return，保存下来，再执行finally的return，覆盖之前的结果，并返回。
### 异常捕获和处理的三种不同方法
- 用try-catch 捕获异常
- 用try-finally 清除异常
- 用try-catch-finally处理所有异常

规则：

    1.try块是必须的，catch块和finally块都是可选的，但必须存在一个或都存在。try块不能单独存在。
    2.try块里的语句运行中出现异常会跳过try块里其他语句，直接运行catch里的语句。
    3.无论try块中是否有异常，无论catch块中的语句是否实现，都会执行finally块里的语句。
    4.如果try块或catch块中有return语句，finally块里的语句会执行在try块或catch块中的return语句前。
    5.如果finally块里有return语句，则直接返回，而不执行try块或catch块里的return语句。
    6.只有一种办法不执行finally块里的语句，那就是调用System.exit(1);方法，即退出java虚拟机。
    
    强调：finally块里的语句在try或catch里的人return前执行！！！
### 异常匹配

抛出异常的时候，异常处理系统会按照代码的书写顺序找出“最近”的处理程序。找到匹配的处理程序之后，它就认为异常将得到处理，然后就不再继续查找。查找的时候并不要求抛出的异常同处理程序所声明的异常完全匹配。派生类的对象也可以匹配其基类的处理程序

---
### Java中访问权限修饰符



![image-20210624104203985](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/image-20210624104203985.png)
---
### 序列化问题

>通常一个类实现序列化方式是实现序列化接口 ***Serializable***
>
>序列化的作用：把数据长久的保存在磁盘中，磁盘和内存是不同的，内存一般在程序运行时占用，数据保存周期短，随程序结束而结束，磁盘可以长久保存数据
>
>transient关键字的作用，在已实现序列化的类中，有的变量不需要保存在磁盘中，就要transient关键字修饰使其在已序列化的类中使变量不序列化
>
>static变量不管加没加transient都不可以被序列化

---
### Java中类的加载顺序 

  (1). 父类静态对象和静态代码块

  (2) 子类静态对象和静态代码块 

  (3) 父类非静态对象和非静态代码块

  (4) 父类构造函数

  (5) 子类  非静态对象和非静态代码块

  (6) 子类构造函数

执行顺序：

    父类静态代码块->子类静态代码块->父类构造代码块->父类构造函数->子类构造代码块->子类构造函数
---
### 继承与构造器

    如果子类构造器没有显示地调用超类的构造器，则将自动地调用超类默认（没有参数）的构造器。
    
    如果超类没有不带参数的构造器，并且在子类的构造器中有没有显示地调用超类的其他构造器，则Java编译器将报告错误。
    
    使用super调用构造器的语句必须是子类构造器的第一条语句。

---
### **switch支持 int及以下（char， short， byte），String， Enum**

**在java 中，声明一个数组时，不能直接限定数组长度，只有在创建实例化对象时，才能对给定数组长度**

**变量不能被defalut修饰**

>native修饰方法，native修饰的方法简单来说就是：一个Java方法调用了一个非Java代码的接口。
>
>定义navtive方法时，并不提供实现体，因为其实现体是用非Java语言在外面实现的。native可以和任何修饰符连用，abstract除外。因为native暗示这个方法时有实现体的，而abstract却显式指明了这个方法没有实现体
>
>当编写一个java源代码文件时，此文件通常被称为编译单元（有时也被称为转译单元）。每个编译单元都必须有一个后缀名.java，而在编译单元内则可以有一个public类，该类的名称必须与文件的名称相同（包括大小写，但不包括文件的后缀名.java）。每个编译单元只能有一个public类，否则编译器就不会接受。如果在该编译单元之中还有额外的类的话，那么在包之外的世界是无法看见这些类的，这是因为它们不是public类，而且它们主要用来为主public类提供支持。          --《Java 编程思想》
>
>​      *注：public类不是必须的，但是如果源文件中有一个（只能有一个）public类的话，文件名必须与这个public类同名，原因*        *是为了方便虚拟机在相应的路径中找到相应的类所对应的字节码文件。所以在没有public类的Java文件中，文件名和类名都没什么联系。**

### Java静态方法不能访问非静态变量
---
### 包机制

>1. 为了更好地组织类，Java提供了包机制。包是类的容器，用于分隔类名空间。如果没有指定包名，所有的示例都属于一个默认的无名包。Java中的包一般均包含相关的类，java是跨平台的，所以java中的包和操作系统没有任何关系，java的包是用来组织文件的一种虚拟文件系统。
>2. import语句并没有将对应的java源文件拷贝到此处仅仅是引入，告诉编译器有使用外部文件，编译的时候要去读取这个外部文件。
>3. 定义在同一个包（package）内的类可以不经过import而直接相互使用。

---
### 强制类型转换

>当多个精度的数字同时进行运算时，最终结果以最高精度为准。在多数情况下，整数和小数的各级混合运算中，一般结果都是double类型的。 
---
# 区别与联系
---
### final/static

>final：
>
>1. final类不能被继承，没有子类，final类中的方法默认是final的。(最终类) 
>
>2. final方法不能被子类的方法覆盖，但可以被继承。 
>
>3. final成员变量表示常量，只能被赋值一次，赋值后值不再改变。 
>
>4. final不能用于修饰构造方法。 
>
>static:
>
>　　static一般表示“静态”的意思，用来修饰成员变量和成员方法，也可以形成静态static代码块
>
>1. static变量
>
>　　先讲一个概念:被static修饰的变量称静态变量或类变量;没有被static修饰的变量称之为实例变量
>
>　　当JVM加载类后,通过类名直接访问静态变量,类的所有实例共享一个静态变量,换个方式讲就是JVM只为静态变量分配一次内存,在加载类的过程中完成;而对于实例变量,每创建一个实例变量都会分配一个内存
>
>2. static方法
>
>　　静态方法只能访问所属类的静态成员变量和方法
>
>  static可以用来修饰内部类,但是不可以用来修饰外部类
>
>3. static代码块
>
>　　静态代码块:类中独立于类成员,JVM加载类的时候按照其先后顺序执行,无需实例化
>
>static final 
>
>1. static final 变量
>
>　　　　属于类的变量且只能赋值一次
>
>2. static final方法
>
>　　　　属于类的方法且不能被重写
>
>3. static final内部类
>
>　　　　属于外部类，且不能被继承
>
>**final定义的成员变量可以在代码块(类变量则静态代码块，实例变量普通代码块)里初始化，也可以在构造器中初始化，也可以在声明变量时初始化。final的局部变量声明时不必马上初始化，但使用时必须初始化，而且只能一次赋值**

---
### 抽象类和接口的区别:

>- 抽象类可以有构造方法，接口中不能有构造方法。
>- 抽象类中可以有普通成员变量，接口中没有普通成员变量，接口中的成员变量都是public static final的，一般用作常量。
>- 抽象类中可以包含静态方法，接口中不能包含静态方法（**在Java SE 8中允许在接口中增加静态方法**）
>- 一个类可以实现多个接口，但只能继承一个抽象类
>- 接口可以被多重实现，抽象类只能被单一继承
>- 如果抽象类实现接口，则可以把接口中方法映射到抽象类中作为抽象方法而不必实现，而在抽象类的子类中实现接口中方法

>抽象类不一定含有抽象方法，接口中的方法都是抽象方法。接口中的方法默认修饰符有public abstract。
>
>一个类只能继承一个一个抽象类，但可以实现多个接口；一个接口可以继承多个接口。Java里类是单继承的，接口是可以多继承的，用关键字extends。
>
>抽象类中的方法是可以有方法体的。JDK1.8之后，接口中的方法也可以有方法体，用default关键字修饰方法。
>
>
>
>抽象类和普通类的主要有三点区别：
>
>1）抽象方法必须为public或者protected（因为如果为private，则不能被子类继承，子类便无法实现该方法），缺省情况下默认为public。
>
>2）抽象类不能用来创建对象；
>
>3）如果一个类继承于一个抽象类，则**子类必须实现父类的抽象方法**。如果子类没有实现父类的抽象方法，则必须将子类也定义为为abstract类。
##### 二者的选用:
    ①.优先选用接口,尽量少用抽象类;
    ②.需要定义子类的行为,又要为子类提供共性功能时才选用抽象类;
---
### “==”和.equals()的区别

- **对象类型不同**

  1、equals()：是超类Object中的方法。

  2、==：是操作符

- **比较的对象不同**

  1、equals()：基本数据类型不能用equals（）方法来比较他们的数值！对于对象的equals（）一共有两种情况：
  1. 对象重写该方法，一般根据自身业务而定比较对象的内容是否相等；
  2. 对象没有重写该方法，仍旧比较的是两对象的地址值，因为在Object的equals（）方法中以＝＝实现。

  2、==：用于比较引用和比较基本数据类型时具有不同的功能，具体如下：

  （1）基础数据类型：比较的是他们的值是否相等，比如两个int类型的变量，比较的是变量的值是否一样。

  （2）引用数据类型：比较的是引用的地址是否相同，比如说新建了两个User对象，比较的是两个User的地址是否一样。

 - **运行速度不同**

   1、equals()：没有'=='运行速度快。

   2、'=='：运行速度比equals()快，因为'=='只是比较引用。
---
### 静态变量与实例变量的区别

>1：首先在语法定义上区别：静态变量前面要加static，实例变量不用
>2：在程序运行时：实例变量输入对象的属性，必须创建了实例对象（如 new）才会被分配空间，才可以使用实例变量，静态变量不属于某个实例对象，而是属于类，也叫类变量，只要程序加载了类的字节码，不用创建任何实例对象就会被分配空间，就可以被使用<br>
>3：总之，实例变量必须创建对象后才可以通过这个对象来使用，静态变量则可以直接使用类名来引用。
---
### 方法重载与方法重写

- 1. 方法重载

     >​		方法重载的定义：同一个类或与他的派生类中，方法名相同，而参数列表不同的方法。其中参数列表不同指的是参数的类型，数量，类型的顺序这三种至少有一种不同。
     >
     >​		方法重载与下列无关：
     >
     >​				与返回值类型无关；与访问修饰符无关
     >
     >​			构造方法也可以重载

  2. 方法重写

     >方法的重写：
     >
     >​		方法的重写的定义：在继承关系的子类中，定义一个与父类相同的方法
     >
     >​		判断是否重写的方式：在方法之前加上@Override
     >
     >方法重写的特点：
     >
     >​			在继承关系的子类中重写父类的方法
     >
     >​			重写的方法必须方法名相同，参数列表也相同
     >
     >​			重写的方法的返回值类型应该与父类中被重写方法的返回值类型相同或是他的子类类型
     >
     >​			重写的方法的访问权限应该与父类中被重写方法的访问权限相同或高于它的访问权限
     >
     >​			重写的方法不能抛出比父类更加宽泛的异常
     >
     >方法重写的注意事项：
     >
     >​				构造方法不能被重写.因为构造方法名必须和类名相同
     >
     >​				private修饰的成员方法不能被重写
     >
     >​				static修饰的方法不能被重写
     >
     >​				final修饰的方法不能被重写
     >
     >​			当子类重写了父类中的方法后，子类对象调用该方法时调用的是子类重写后的方法
     
     **只要是被子类重写的方法，不被super调用都是调用子类方法**
     
     *重载就是一句话：同名不同参，返回值无关。
覆盖/重写：同名同参*
---
### 基本数据类型和包装类的比较

>值类型“==”比较的是具体的值，引用类型“==”比较的是地址。 
>
>1. int与Integer在进行比较的时候，Integer会自动进行拆箱，转换为int与int进行比较，比较的是具体的值。 
>
>2. Integer与Integer比较的时候，由于直接赋值的时候会进行自动装箱（ Integer a = 1 或 Integer  a = Integer.valueOf(1) ）,这里需要注意两个问题： 
>
>   	① 当整数范围在-128到127之间时，不需要创建新的Integer对象，而是从缓存（IntegerCache）中获取已经创建好的Integer对象。 
>   	  	  	  	  	
>   	②当大于或小于这个范围的时候，就会直接调用new Integer来创建Integer对象。 
>
>​       所以Integer a = 128 与 Integer b = Integer.valueOf(128)  都创建了新的Integer对象，用“==”比较会返回false 
>
>3. Integer a = new Integer(1)和Integer b =  1不同是因为前者会创建对象，存储在堆中，而后者因为在-128到127的范围内，不会创建新的对象，而是从IntegerCache中获取。两者地址不同，用“==”比较自然会返回false。

​	①无论如何，**Integer与new Integer不会相等。不会经历拆箱过程**，
​    ②***两个都是非new出来的Integer，如果数在-128到127之间，则是true,否则为false*** Cache
  					java在编译Integer i2 = 128的时候,被翻译成-> Integer i2 = Integer.valueOf(128);而valueOf()函数会对-128到127之间的数进行缓存
​    ③**两个都是new出来的,都为false**
​    ④**int和integer(无论new否)比，都为true，因为会把Integer自动拆箱为int再去比**

通常 new 出来的对象都是保存在堆内存中的，以此方式创建小而简单的变量往往是不划算的。所以对于这些基本类型的创建方法，Java 使用了和 C/C++ 一样的策略。也就是说，不是使用 new 创建变量，而是使用一个“自动”变量。 这个变量直接存储”值”，并置于栈内存中，因此更加高效。基本类型有自己对应的包装类型，如果你希望在堆内存里表示基本类型的数据，就需要用到它们的包装类。

---
### 处理流和节点流

>按照流是否直接与特定的地方（如磁盘、内存、设备等）相连，分为节点流和处理流两类。
>
>- 节点流：可以从或向一个特定的地方（节点）读写数据。如FileReader.
>- 处理流：是对一个已存在的流的连接和封装，通过所封装的流的功能调用实现数据读写。如BufferedReader.处理流的构造方法总是要带一个其他的流对象做参数。一个流对象经过其他流的多次包装，称为流的链接。
>
>**JAVA常用的节点流：**
>
>- 文 件 FileInputStream FileOutputStrean FileReader FileWriter 文件进行处理的节点流。
>- 字符串 StringReader StringWriter 对字符串进行处理的节点流。
>- 数 组 ByteArrayInputStream ByteArrayOutputStreamCharArrayReader CharArrayWriter 对数组进行处理的节点流（对应的不再是文件，而是内存中的一个数组）。
>- 管 道 PipedInputStream PipedOutputStream PipedReaderPipedWriter对管道进行处理的节点流。
>
>**常用处理流（关闭处理流使用关闭里面的节点流）**
>
>- 缓冲流：BufferedInputStrean BufferedOutputStream BufferedReader BufferedWriter 增加缓冲功能，避免频繁读写硬盘。
>
>- 转换流：InputStreamReader OutputStreamReader 实现字节流和字符流之间的转换。
>- 数据流 DataInputStream DataOutputStream 等-提供将基础数据类型写入到文件中，或者读取出来.
>
>### 流的关闭顺序
>
>1. 一般情况下是：先打开的后关闭，后打开的先关闭
>2. 另一种情况：看依赖关系，如果流a依赖流b，应该先关闭流a，再关闭流b。例如，处理流a依赖节点流b，应该先关闭处理流a，再关闭节点流b
>3. 可以只关闭处理流，不用关闭节点流。处理流关闭的时候，会调用其处理的节点流的关闭方法。

---
### String/StringBuffer/StringBulider比较
---
1. 从数据来看:
   - String 数据不可变
   - StringBuffer / StringBuilder 数据可变
2. 从内存来看:
   - String 浪费内存
   - StringBuffer / StringBuilder 始终都是同一个对象
3. 从线程安全来看:
   - String / StringBuffer  线程安全
   - StringBuilder  线程不安全
4. 从效率来看
   - StringBuilder > String >StringBuffer
---
### Object的方法解析

方法 |	描述 
---|---
protected Object clone()|创建并返回当前对象的一份拷贝
protected void finalize()|	当垃圾回收器判断出该对象不再被引用时，就会调用finalize()方法。在对象的创建与销毁中有对finalizers的介绍。
boolean equals(Object obj)|	判断另外一个对象是否与当前对象相等
int hasCode()|	返回当前对象的哈希值
String toString()	|返回一个表示当前对象的字符串
void notify()|	唤醒一个等待当前对象的锁监视器的线程。我们将会在第9篇文章并发最佳实践中详细介绍此方法
void notifyAll()|	唤醒所有等待当前对象的锁监视器的线程。我们将会在第9篇文章并发最佳实践中详细介绍此方法
void wait()<br>void wait(long timeout)<br>void wait(long timeout, int nanos)|	使当前线程进入等待状态直到其他线程调用了当前对象的notify()或notifyAll()方法。
---
### 介绍快速排序算法,并指出其时间复杂度
---

### 给对象开辟内存空间后，将相应的地址赋值给引用，有几种方式
---
### 什么是多态

    多态是同一个行为具有多个不同表现形式或形态的能力。
    多态就是同一个接口，使用不同的实例而执行不同操作
    多态的前提: 继承
    多态的作用: 提高代码拓展性
    多态的体现:线上转型
    多态的限制:向上转型发生时,子类独有的成员无法使用
---
### java位操作
    按位或|
    按位且&
    按位取反~
    按位异或^
---
### 构造方法相关:
    构造方法是一种特殊的方法，具有以下特点。
    （1）构造方法的方法名必须与类名相同。
    （2）构造方法没有返回类型，也不能定义为void，在方法名前面不声明方法类型。
    （3）构造方法的主要作用是完成对象的初始化工作，它能够把定义对象时的参数传给对象的域。
    （4）一个类可以定义多个构造方法，如果在定义类时没有定义构造方法，则编译系统会自动插入一个无参数的默认构造器，这个构造器不执行任何代码。
    （5）构造方法可以重载，以参数的个数，类型，顺序
---
### Integer和String的互相转换
1. 如何将字串 String 转换成整数 int?
   

有两个方法:
```java
 int i = Integer.parseInt([String])
 int i = Integer.valueOf(my_str).intValue();.
```
2. 如何将整数 int 转换成字串 String ?

有叁种方法:
```java
1、String s = String.valueOf(i);
2、String s = Integer.toString(i);
3、String s = "" + i;

```
---

### 什么情况下会发生空指针异常
---
    当一个变量的值为null 时，在Java里面表示一个不存在的空对象，没有实际内容，没有给它分配内存，null 也是对象成员变量的默认值。
    所以，一个对象如果没有进行初始化操作，这时候，如果你调用这个对象的方法或者变量，就会出现空指针异常。
### 什么是序列化?
---
    序列化：把对象转化为可传输的字节序列过程称为序列化。
    反序列化：把字节序列还原为对象的过程称为反序列化。

序列化最终的目的是为了对象可以跨平台存储，和进行网络传输,把对象转成字节数组的时候就制定一种规则（序列化），那么我们从IO流里面读出数据的时候再以这种规则把对象还原回来（反序列化）

**static 属性不能被序列化/Transient 属性不会被序列化**

序列化版本号serialVersionUID:
    所有实现序列化的对象都必须要有个版本号，这个版本号可以由我们自己定义，当我们没定义的时候JDK工具会按照我们对象的属性生成一个对应的版本号
### Statement 和 PreparedStatement 的区别
---
1. PreparedStatement(预编译sql) 是 Statement(拼接sql) 的子接口
2. PreparedStatement参数用?占位符标识,对 sql 进行预编译.可以防止sql注入;Statement 的sql参数是拼接Sql 有sql注入的风险,在开发中常用PreparedStatement
3. PreparedStatement 对预编译的sql的执行代码进行缓存,对于重复调用的sql不许要重新编译,能最大可能提高性能
4. PreparedStatement 支持对Blob类型的数据进行操作,而 Statement 不能
---

### 调优HashMap

> 容量（Capacity）：表中存储的桶数量。
> 初始容量（Initial Capacity）：当表被创建时，桶的初始个数。 HashMap 和 HashSet 有可以让你指定初始容量的构造器。<br>
> 个数（Size）：目前存储在表中的键值对的个数。<br>
> 负载因子（Load factor）：通常表现为 $\frac{size}{capacity}$。当负载因子大小为 0 的时候表示为一个空表。当负载因子大小为 0.5 表示为一个半满表（half-full table），以此类推。轻负载的表几乎没有冲突，因此是插入和查找的最佳选择（但会减慢使用迭代器进行遍历的过程）。<br>
> HashMap 和 HashSet 有可以让你指定负载因子的构造器。当表内容量达到了负载因子，集合就会自动扩充为原始容量（桶的数量）的两倍，并且会将原始的对象存储在新的桶集合中（也被称为 rehashing）<br>
> HashMap 中负载因子的大小为 0.75（当表内容量大小不足四分之三的时候，不会发生 rehashing 现象）。这看起来是一个非常好的同时考虑到时间和空间消耗的平衡策略。更高的负载因子会减少空间的消耗，但是会增加查询的耗时。重要的是，查询操作是你使用的最频繁的一个操作（包括 get() 和 put() 方法）。<br>
> 如果你知道存储在 HashMap 中确切的条目个数，直接创建一个足够容量大小的 HashMap，以避免自动发生的 rehashing 操作。提高系统的性能



### Vector和Collections.synchronizedList()的区别
1. Vector使用同步方法实现，synchronizedList使用同步代码块实现
2. 两者的扩充数组容量方式不一样
3. 使用SynchronizedList的时候，进行遍历时要手动进行同步处理。 
4. SynchronizedList可以指定锁定的对象。
5. SynchronizedList有很好的扩展和兼容功能。他可以将所有的List的子类转成线程安全的类。
---

### 同步代码块和同步方法的区别
1. 同步代码块在锁定的范围上可能比同步方法要小，一般来说锁的范围大小和性能是成反比的。 
2. 同步块可以更加精确的控制锁的作用域（锁的作用域就是从锁被获取到其被释放的时间），同步方法的锁的作用域就是整个方法。
3. 静态代码块可以选择对哪个对象加锁，但是静态方法只能给this对象加锁。

---
### Java8时间API    
```java
  /**
     * Java8+ 新日期类
     * SimpleDataFormatter ----> 就时间API 线程不安全
     * DataTimeFormatter ---> 1.8 提供时间API 线程安全
     * <p>
     * 1. LocalDate(日期) LocalTime(时间) LocalDataTime(日期加时间)
     * 2. Instant : 时间戳(以Unix元年到现在的毫秒值)
     * 3. Duration : 计算两个时间之间的间隔
     *    Period: 计算两个日期之间的间隔
     * 4. TemporalAdjuster : 时间校正器 对时间进行运算  with...  指定时间
     *    TemporalAdjusters: 该类通过静态方法提供了大量的常用实现
     * 5. DateTimeFormatter: 格式化时间/日期
     */
    @org.junit.jupiter.api.Test
    void dateDemo() throws InterruptedException {
        Set<String> availableZoneIds = ZoneId.getAvailableZoneIds();
        for (String zoneId : availableZoneIds) {
            log.info("zoneId:{}",zoneId);
        }
        DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ISO_DATE;
        LocalDateTime localDateTimer = LocalDateTime.now();
        String format = localDateTimer.format(dateTimeFormatter);
        log.info("格式化后的日期格式: {}",format);
        DateTimeFormatter dateTimeFormatterDIY = DateTimeFormatter.ofPattern("yyyy年MM月dd日 HH:mm:ss");
        String formatDIY = localDateTimer.format(dateTimeFormatterDIY);
        log.info("格式化后的日期格式: {}",formatDIY);
        LocalDateTime time = LocalDateTime.parse(formatDIY, dateTimeFormatterDIY);
        log.info("字符串转data:{}",time);
        Instant now = Instant.now();
        Thread.sleep(1L);
        Instant now1 = Instant.now();
        long nano = Duration.between(now, now1).toMillis();
        log.info("程序睡眠{}纳秒",nano);
        LocalDateTime localDateTime = LocalDateTime.now();
        log.info(localDateTime);
        LocalDateTime dateTime = LocalDateTime.of(2021, 12, 12, 23, 23);
        log.info(dateTime);
        LocalDateTime plusYears = localDateTime.plusYears(2);
        log.info(plusYears);
        LocalDateTime minusMonths = localDateTime.minusMonths(2);
        log.info(minusMonths);

        log.info(localDateTime.getYear());
        log.info(localDateTime.getMonthValue());
        log.info(localDateTime.getDayOfMonth());
        log.info(localDateTime.getHour());
        log.info(localDateTime.getMinute());
        log.info(localDateTime.getSecond());
        log.info(localDateTime.getDayOfWeek());
        log.info(localDateTime.getDayOfYear());
        log.info(Instant.now());
    }
```

#### 创建流
```java
   /**
     * 1. 创建Stream
     * 2. 中间操作(操作流不会改变源数据)
     * 3. 终止操作
     */
    @Test
    void testStream(){
        // 1. 通过Collection 系列集合提供的 stream() 或 parallelStream()
        List<String> stringList = new ArrayList<>();
        Stream<String> stream = stringList.stream();
        // 2. 通过Arrays中的静态方法 stream() 获取数组流
        Stream<String> stringStream = Arrays.stream(new String[]{"1", "2", "3"});
        // 3. 通过Stream类中的静态方法of()
        Stream<Integer> integerStream = Stream.of(1, 23, 34, 5);
        // 4. 创建无限流
        // 迭代
        Stream<Integer> iterate = Stream.iterate(0, x -> x + 2);
        iterate.limit(10).forEach(log::info);
        //生成
        Stream.generate(() -> Math.random()).limit(5).forEach(log::info);
    }
```
#### 中间操作





### Java8内置的四大核心函数式接口
>1. Consumer<T> :消费型接口
>      void accept(T t)
>2. Supplier<T> : 供给型接口
>      T get();
>3. Function<T,R> :函数型接口
>      R apply(T t)
>4. Predicate<T> : 断言型接口
>      boolean test(T t)

---

### 名词解析

> 静态属性: static开头定义的属性
> 静态方法块: static{}包起来的代码块
> 普通属性: 非static定义的属性
> 普通方法块: {}包起来的代码块
> 构造函数: 类名相同的方法
> 方法: 普通方法

执行顺序:
静态属性初始化 ---> 静态方法块初始化 ---> 普通属性初始化 ---> 普通方法块初始化 ---> 构造函数初始化

---

### Collections

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/20221207141009.png)



![image-20221207141347131](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/image-20221207141347131.png)

![image-20221207141424284](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/image-20221207141424284.png)

![image-20221207141450592](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/image-20221207141450592.png)

![image-20221207141516750](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/javabasic/image-20221207141516750.png)
