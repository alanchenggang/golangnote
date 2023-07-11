## GO学习笔记

### 字符串
#### Q1. 什么是string
字符串的本质是**不可修改的byte数组**
#### Q2. string 的遍历

```go
func LearnString() {
	s := "GOLANG"
	fmt.Println(s)
	for i, ele := range s {
		fmt.Printf("%d %c\n", i, ele)
	}
	// s[0] = `f`   cannot assign to s[0] (value of type byte)
	fmt.Println(len(s)) // 6
	a := "golang你好"
	// 一般来说 一个汉字占3个字节
	fmt.Println(len(a))   // 12
	arr := []rune(a)      // 将string转换为rune切片 可以求得正确的长度
	fmt.Println(len(arr)) // 8
	for _, ele := range a {
		fmt.Printf("%c\n", ele)
	}
}
```

#### Q3. string 的拼接

```go
	// 字符串的拼接
	// 1. 使用+号 
	c := s + a
	fmt.Println(c)
	// 2. 使用fmt.Sprintf
	d := fmt.Sprint(s, a)
	fmt.Println(d)
	// 3. 使用strings.Join
	e := strings.Join([]string{s, a}, "")
	fmt.Println(e)
```



![image-20230620113724079](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230620113724079.png)

### 指针


#### Q1. 什么是指针

>在计算机科学中，指针（Pointer）是一种变量，它存储了一个内存地址的值。简单来说，指针是一个用于存储内存地址的变量。这个地址可以是任何数据（如整数、字符、数组、对象等）在内存中的位置。
>
>通过使用指针，可以实现对内存中数据的间接访问。指针可以用来传递数据、修改数据，以及动态分配和释放内存等。在许多编程语言中，特别是低级语言如 C 和 C++，指针是一项重要的概念。
>
>指针通常包括两个部分：地址和类型。地址是内存位置的值，类型表示该地址上存储的数据的类型。指针的操作包括获取地址（取址操作符 &）、获取地址上的值（间接引用操作符 *）以及指针的赋值和比较等。
>
>指针的使用需要小心谨慎，因为错误的指针操作可能会导致程序崩溃或产生难以调试的错误。正确地使用指针可以提高程序的效率和灵活性，但也需要程序员对内存管理有一定的了解和责任。

####  Q2. Go 的指针和 C 语言的指针有什么区别

1. 指针声明和操作符：在 C 语言中，使用星号（*）来声明指针类型和进行间接引用操作，而在 Go 语言中，指针类型使用 `*` 进行声明，但间接引用操作使用`.` 运算符，而不是 `*`
2. 空指针：在 C 语言中，可以将指针设置为 NULL，表示指针不指向任何有效的内存地址。而在 Go 语言中，使用 `nil` 表示空指针
3. 内存管理：在 C 语言中，程序员负责手动分配和释放内存，使用 `malloc()` 和 `free()` 等函数。而在 Go 语言中，有垃圾回收器负责自动管理内存，程序员无需手动释放内存。这种自动内存管理可以减少内存泄漏和悬挂指针等问题
4. 指针运算：在 C 语言中，可以对指针进行算术运算，例如指针的加法和减法。而在 Go 语言中，指针运算是受限的，不允许进行指针的算术运算(Go保证安全性)
5. 安全性：Go 语言在设计上注重安全性和简洁性，对指针的使用进行了限制，避免了一些常见的错误和安全隐患，如空指针引用和悬挂指针等。这使得 Go 语言相对于 C 语言更容易编写安全和可靠的代码

总体而言，Go 语言对指针的处理更加简化和安全，提供了更高级别的内存管理和类型安全性，而 C 语言则更加底层和灵活，允许更多的指针操作和内存控制。

---

### 函数

函数本质是一个指针 指向其函数的内存地址

#### Q1. 匿名函数:

```go
//匿名函数
	var getSum = func(n1, n2 int) (sum, difference int) {
		sum = n1 + n2
		difference = n1 - n2
		return
	}
	// 立即调用的匿名函数
	res3, res4 := func(n1, n2 int) (sum, difference int) {
		sum = n1 + n2
		difference = n1 - n2
		return
	}(2, 3)
```

#### Q2. defer: 延迟执行函数

1. 延迟执行的函数会被压入一个栈中 return之后按照先进后出的顺序调用

2. 当defer后接函数有参数时其参数会立即求值

3. defer常用于资源释放、文件关闭、解锁以及记录时间等操作

```go
   // 闭包
   func deferUtil() func(int) int {
   	i := 0
   	return func(n int) int {
   		fmt.Printf("本次调用的参数为%v\n", n)
   		i++
   		fmt.Printf("deferUtil被第%v次调用\n", i)
   		return i
   	}
   }
   
   // Defer  延迟执行函数
   func Defer() int {
   	f := deferUtil()
   	defer f(f(3)) // defer的注册
   	return f(2)
   }
```

![image-20230620120558487](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230620120558487.png)

#### Q3. recover 错误捕捉 函数能防止异常错误退出:

```go
func DeferRecover() {
	defer func() {
		err := recover()
		if err != nil {
			fmt.Println("err=", err)
		}
	}()
	n := 0
	fmt.Println(3 / n)
}
```
#### Q4.  什么是闭包

>Go 语言支持函数闭包（Function Closure）。闭包是指一个函数值（函数变量）捕获并绑定了其周围的函数体内的变量。这意味着闭包函数可以访问并操作其外部函数中声明的变量，即使外部函数已经返回，闭包仍然可以使用这些变量。
>
>简言之:
>
>当我们谈论闭包时，可以将其想象成一个包裹（closure），里面装着一个函数和一些与函数相关的数据。这个函数可以访问并操作这些数据，即使这些数据在函数外部也是可见的。
>
>闭包通常用于在函数内部创建函数，并且这些函数可以访问函数外部的变量。这就像是在函数内部构建了一个小的「记忆」，可以保存一些数据状态。每当我们调用闭包时，它会记住之前的状态，并在后续调用中保持这些状态的连续性。

闭包在 Go 语言中具有以下特点：

1. 函数变量：在 Go 语言中，函数可以作为一种类型，可以被赋值给变量，作为参数传递给其他函数或者作为返回值。通过将函数赋值给变量，我们可以创建闭包。

2. 引用外部变量：闭包函数可以访问和使用其外部函数中声明的变量，包括函数参数和局部变量。闭包通过引用捕获这些变量，因此即使外部函数执行完毕，闭包仍然可以使用这些变量。

3. 保持状态：闭包函数可以在不同的时间点被调用，而它所捕获的变量将保持其最新的状态。这使得闭包可以用于实现保持状态或者记忆的功能，例如计数器或者延迟执行。

   ```go
   func counter() func() int {
       count := 0
       return func() int {
           count++
           return count
       }
   }
   
   func main() {
       c := counter()
       fmt.Println(c()) // 输出: 1
       fmt.Println(c()) // 输出: 2
       fmt.Println(c()) // 输出: 3
   }
   ```

   在这个例子中，我们定义了一个 `counter` 函数，它返回一个匿名函数。这个匿名函数引用了外部函数中的 `count` 变量。每次调用 `counter` 函数时，都会创建一个新的闭包实例，并且每个实例都保留了自己的 `count` 变量。每次调用闭包函数时，它会增加 `count` 的值并返回。

   闭包的妙处在于它可以记住之前的状态。在这个例子中，无论我们调用多少次闭包函数，它都会记住每次调用的计数值，并保持连续增加。这是因为**闭包函数持有对外部变量的引用**，而不是简单地创建新的变量副本。





### 接口

#### Q1. go中接口是什么



### 数组



### Map

#### Q1: map是什么

在Go语言中，map是一种无序的键值对的集合。它是一种引用类型，可以像其他引用类型一样通过make函数进行初始化。map的键可以是任何可比较类型，例如字符串、数字、结构体等，而值可以是任何类型。map的底层实现是哈希表(类似Java的HashMap?)，因此可以在O(1)时间复杂度内完成插入、查找和删除操作。如果需要使用map，请确保在使用前进行初始化

> Java的HashMap 和Golang的map的异同
>
> Java的HashMap和Go的map都是键值对的集合，但是它们在实现上有一些不同。Java的HashMap是基于哈希表实现的，而Go的map也是基于哈希表实现的。但是，Java的HashMap允许键和值为null，而Go的map不允许键为nil。此外，Java的HashMap是线程不安全的，需要使用Collections.synchronizedMap()方法或者ConcurrentHashMap来保证线程安全，而Go的map是并发安全的，可以在多个goroutine中同时使用。

#### Q2: map的增删改查和遍历

```go
// Map 键值对
func Map() {

	var m0 map[string]string
	fmt.Println(m0)
	m0 = make(map[string]string, 16)
	fmt.Println(m0)
	m1 := map[string]string{
		"name": "张三",
		"age":  "18",
		"id":   strings.ReplaceAll(uuid.New().String(), "-", ""), // 生成uuid
	}
	fmt.Println(m1)

	// map的插入
	m1["password"] = "123"
	fmt.Println(m1)
	// 修改
	m1["age"] = "100"
	fmt.Println(m1)

	// 查找
	v, ok := m1["name1"]
	if ok {
		fmt.Println(v)
	} else {
		fmt.Println("不存在")
	}

	// map的遍历
	for key, value := range m1 {
		fmt.Println(key, "=", value)
	}
	// 删除 通过delete(map,key)函数删除单个元素
	// 删除不存在的元素也不会报错
	delete(m1, "name")
	fmt.Println(m1)
	// 删除所有元素 通过make()函数重新make一次 或者指向nil
	m1 = nil
	m2 := make(map[string]string, 16)
	fmt.Println(m1)
	fmt.Println(m2)
}
```



![image-20230620114318713](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230620114318713.png)

### 结构体

###  切片

#### Q1: 切片是什么
在go的源文件中,切片底层是一个结构体

```go
type slice struct {
	array unsafe.Pointer // 指向底层数组
	len   int  // 底层数组的长度
	cap   int  // 底层数组的长度
}
```

#### Q2: 切片的赋值/拷贝

当 brr:=arr时,会拷贝slice1给slice2,但是他们Pointer 指针指向的是同一个底层数组 因此操作切片2的同时切片1的内容也会发生变化

![image-20230620105124794](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230620105124794.png)

对切片的修改结果会相互影响

```go
func AppendDemo() {
	arr := make([]int, 3, 5)
	arr[0], arr[1], arr[2] = 2, 7, 9
	//底层数组 : 2, 7, 9, 8
	brr := append(arr, 8)
	// brr和arr共享底层数组 因此修改arr会影响brr 反之亦然
	fmt.Println(len(arr))
	fmt.Println(len(brr)) 
	//底层数组 : 2, 7, 9, 10
	arr = append(arr, 10)
	fmt.Println(arr)
	fmt.Println(brr)
}
```

![image-20230620110146241](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230620110146241.png)

当切片扩容后.对切片的修改则不会再相互影响

```go
func AppendDemo() {
	arr := make([]int, 3, 5)
	arr[0], arr[1], arr[2] = 2, 7, 9
	//底层数组 : 2, 7, 9, 8
	brr := append(arr, 8)
	// brr和arr共享底层数组 因此修改arr会影响brr 反之亦然
	fmt.Println(len(arr)) // 3
	fmt.Println(len(brr)) // 4
	//底层数组 : 2, 7, 9, 10
	arr = append(arr, 10)
	fmt.Println(arr)
	fmt.Println(brr)
	fmt.Println("brr 插入 	容量满")
	brr = append(brr, 15)
	arr[0] = 100
	brr[1] = 200
	fmt.Println("容量满 arr", arr) //容量满 arr [100 200 9 10]
	fmt.Println("容量满 brr", brr) //容量满 brr [100 200 9 10 15]
	//根据以上运行结果可知 当容量未满时，arr和brr始终共享底层数组

	fmt.Println("brr 插入  需要扩容")
	brr = append(brr, 16)
	arr[2] = 200
	brr[3] = 300
	fmt.Println("需要扩容 arr", arr) //需要扩容 arr [100 200 200 10]
	fmt.Println("需要扩容 brr", brr) //需要扩容 brr [100 200 9 300 15 16]
	// 根据以上运行结果可知 当容量满时，arr和brr不再共享底层数组
	// 底层会重新分配一个数组，将原数组的值拷贝到新数组中 将新数组的地址赋值给brr的Pointer 从而对brr的修改不会影响arr

}
```

![image-20230620111647876](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230620111647876.png)

#### Q3. 切片的遍历

同数组遍历一致 可使用普通for循环或者for...range 遍历



### 时间操作

```go
const (
	DATE = "2006-01-02"          // 代表年月日
	TIME = "2006-01-02 15:04:05" // 代表年月日时分秒
)

// 时间常见操作
func PackageTime() {
	// 获取当前时间
	now := time.Now()
	fmt.Println(now.Unix())

	fmt.Println()
	//休眠
	for i := 0; i < 5; i++ {
		fmt.Print(".")
		time.Sleep(time.Millisecond * 2)
	}
	fmt.Println()
	// 两个时刻之差
	t1 := time.Now()
	diff := t1.Sub(now)
	fmt.Println(diff.Milliseconds())            // 获取时间差毫秒数
	fmt.Println(time.Since(now).Milliseconds()) // 获取时间差

	// 计算时间的加法
	// time + duration = time
	d := 2 * time.Second
	t2 := t1.Add(d)
	fmt.Println(t2.Unix())

	// 时间转换
	duration, err := time.ParseDuration("1000s")
	if err != nil {
		panic(err)
	}
	fmt.Println(duration)

	// 时间的格式化
	fmt.Println(now.Format(DATE)) //2023-06-20
	s := now.Format(TIME)
	fmt.Println(s) //2023-06-20 17:14:34

	// 时间的解析
	loc, _ := time.LoadLocation("Asia/Shanghai")	// 构造时区
	t3, _ := time.ParseInLocation(TIME, s, loc)
	fmt.Println(t3.Unix())

}

```

![image-20230620172121482](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230620172121482.png)

### 读写文件操作



#### 文件访问权限

![image-20230703185136299](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230703185136299.png)



![image-20230703185258704](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230703185258704.png)

```go
// 文件操作
// 判断文件是否存在
func fileExist(path string) bool {
	fileInfo, err := os.Stat(path)
	if err == nil {
		return !fileInfo.IsDir()
	}
	return os.IsExist(err)
}

// 根据文件路径创建文件夹
func createDir(path string) error {
	paths := strings.Split(path, "/")
	fmt.Println(paths)
	var dirPath = ""
	for i, v := range paths {
		if i == len(paths)-1 {
			break
		}
		if i != 0 {
			dirPath += "/"
		}
		dirPath += v
	}
	fmt.Println(dirPath)
	return os.MkdirAll(dirPath, os.ModePerm)
}
func FileOperation() {
	//dir := readDir("D:\\CodeProjects")
	//fmt.Println(dir)
	fileRW("d1.txt")
}

// 读取文件夹下的所有文件
func readDir(path string) []string {
	var file = make([]string, 16)
	dirEntries, err := os.ReadDir(path)
	if err != nil {
		panic(err)
	}
	for _, entry := range dirEntries {
		file = append(file, entry.Name())
	}
	return file
}

// 文件读写
func fileRW(path string) {
	// 打开文件
	file, err := os.OpenFile(path, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0644) //os.O_TRUNC 清空文件 os.O_APPEND 追加写入
	if err != nil {
		panic(err)
	}
	// 关闭文件
	defer func(file *os.File) {
		err := file.Close()
		if err != nil {
			panic(err)
		}
	}(file)

	//读文件
	readByteSlice := make([]byte, 1024)
	read, err := file.Read(readByteSlice)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(readByteSlice[:read]))
	//写入文件
	context := "大多大事单倍行距阿萨德大萨达是多少收到是是"
	write, err := file.WriteString(context)
	if err != nil {
		panic(err)
	}
	fmt.Println(write) // 成功向文件中写入了多少字节

	//writer := bufio.NewWriter(file)
	//_, err = writer.WriteString("hello world")
	//if err != nil {
	//	panic(err)
	//}
	// 刷新缓冲区
	//err = writer.Flush()
	//if err != nil {
	//	panic(err)
	//}
}
```

**改进: 使用缓冲区优化文件读写**

```go
unc fileRWBuf(path string) {
	// 打开文件
	file, err := os.Open(path)
	if err != nil {
		panic(err)
	}
	// 延时关闭文件 (必须有)
	defer func(file *os.File) {
		err := file.Close()
		if err != nil {
			panic(err)
		}
	}(file)

	//使用缓冲区读取文件 减少磁盘IO提高程序性能
	reader := bufio.NewReader(file)
	for {
		line, err := reader.ReadString('\n') // 碰到换行符就结束读取

		if err != nil {
			// 判断是否读取到文件末尾
			if err == io.EOF {
				fmt.Println(line)
				fmt.Println("文件读取完毕")
				break
			}
			panic(err)
		}
		fmt.Print(line)
	}

	//使用缓冲区写入文件 减少磁盘IO提高程序性能
	writer := bufio.NewWriter(file)
	_, err = writer.WriteString("hello world")
	if err != nil {
		panic(err)
	}
	//刷新缓冲区
	err = writer.Flush()
	if err != nil {
		panic(err)
	}
	
}
```

### JSON序列化

```go
import (
	"fmt"
	"github.com/bytedance/sonic"
)

type Student struct {
	Name   string `json:"name"`
	Age    int    `json:"age"`
	Gender bool   `json:"gender"`
}
type Class struct {
	Id       string    `json:"id"`
	Students []Student `json:"students slice"`
}

func SerialJSON() {
	s :=
		Student{"张宝磊", 58, false}
	c := Class{
		Id:       "1(1)班",
		Students: []Student{s, s, s},
	}
	// json 序列化 通过json.Marshal()函数将一个Go语言中的结构体转换为JSON格式的字符串
	bytes1, err1 := sonic.Marshal(c)
	if err1 != nil {
		panic(err1)
	}
	//for _, v := range bytes {
	//	fmt.Printf("%c", v)
	//}
	str1 := string(bytes1)
	fmt.Println(str1)

	// json 反序列化 通过json.Unmarshal()函数将一个JSON格式的字符串转换为Go语言中的结构体
	var c2 Class
	err1 = sonic.Unmarshal(bytes1, &c2)
	if err1 != nil {
		panic(err1)
	}
	fmt.Printf("%v", c2)
	// 字节跳动sonic性能更佳
	// sonic 序列化 sonic.Marshal()函数将一个Go语言中的结构体转换为JSON格式的字符串
	bytes, err := sonic.Marshal(c)
	if err != nil {
		panic(err)
	}
	//for _, v := range bytes {
	//	fmt.Printf("%c", v)
	//}
	str := string(bytes)
	fmt.Println(str)

	// sonic 反序列化 sonic.Unmarshal()函数将一个JSON格式的字符串转换为Go语言中的结构体
	var c1 Class
	err = sonic.Unmarshal(bytes, &c1)
	if err != nil {
		panic(err)
	}
	fmt.Printf("%v", c1)

}

```

![image-20230620185306349](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230620185306349.png)

### 单元测试&&基准测试

1. 单元测试/基准测试文件名通常以 被测试文件名_test.go 命名
2. 单元测试函数名必须以Test开头 函数参数为 t *testing.T 例如: <code>func TestSerialSonic(t *testing.T) </code>
3. 基准测试函数名必须以Benchmark开头 函数参数问 b *testing.B 例如: <code>func BenchmarkSerialSonic(b *testing.B)  </code>

```go
package std

import (
	"encoding/json"
	"fmt"
	"github.com/bytedance/sonic"
	"testing"
)

// 单元测试
var (
	s = Student{"张宝磊", 58, false}
	c = Class{
		Id:       "1(1)班",
		Students: []Student{s, s, s},
	}
)

// TestSerialJSON 单元测试
func TestSerialJSON(t *testing.T) {
	marshal, err := json.Marshal(c)
	if err != nil {
		t.Fail()
	}
	var c2 Class
	err = json.Unmarshal(marshal, &c2)
	if err != nil {
		t.Fail()
	}
	if !(c.Id == c2.Id && len(c.Students) == len(c2.Students)) {
		t.Fail()
	}

	fmt.Println("json 没毛病")
}
// TestSerialSonic 单元测试
func TestSerialSonic(t *testing.T) {
	marshal, err := sonic.Marshal(c)
	if err != nil {
		t.Fail()
	}
	var c2 Class
	err = sonic.Unmarshal(marshal, &c2)
	if err != nil {
		t.Fail()
	}
	if !(c.Id == c2.Id && len(c.Students) == len(c2.Students)) {
		t.Fail()
	}

	fmt.Println("sonic 没毛病")
}

// TestSerialJSON 基准测试 
func BenchmarkSerialJSON(b *testing.B) {
	for i := 0; i < b.N; i++ {
		marshal, _ := json.Marshal(c)
		var c2 Class
		_ = json.Unmarshal(marshal, &c2)
	}
}

// BenchmarkSerialSonic 基准测试
func BenchmarkSerialSonic(b *testing.B) {
	for i := 0; i < b.N; i++ {
		marshal, _ := sonic.Marshal(c)
		var c2 Class
		_ = sonic.Unmarshal(marshal, &c2)
	}
}
```

基准测试结果:

![image-20230620192644475](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230620192644475.png)

![image-20230620192650101](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230620192650101.png)



### Channel

#### Q1. 管道是什么?底层由什么数据结构实现

![image-20230621165140089](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230621165140089.png)

在Go语言中，管道（channel）是一种用于在多个goroutine之间进行通信和同步的机制。管道可以看作是一种特殊的队列，它具有先进先出的特性，但是与普通的队列不同的是，管道可以在多个goroutine之间进行安全的数据传输。

管道的底层实现是一个带锁的队列(环形队列,FIFO)，它使用了类似于链表的数据结构来存储数据。当一个goroutine向管道中写入数据时，数据会被添加到队列的尾部；而当一个goroutine从管道中读取数据时，数据会从队列的头部被取出。由于管道是带锁的，因此在多个goroutine同时访问管道时，可以保证数据的安全性和正确性。

---

#### Q2. go中channel的作用

1. 协程间数据的传递

   ```go
   func Channel() {
   	// 创建管道
   	ch := make(chan int, 1)
   	fmt.Println(time.Now().Unix())
   	// 从管道中读取数据
   	go func() {
   		a := <-ch
   		fmt.Printf("%d 读出%d\n", time.Now().Unix(), a)
   	}()
   	time.Sleep(2 * time.Second)
   	// 向管道中写入数据
   	go func() {
   		ch <- 4
   		fmt.Println(time.Now().Unix(), "写入channel完成")
   	}()
   
   	time.Sleep(1 * time.Second)
   
   }
   ```

   

2. 协程间协调同步

```go
// 生产者消费者
// 目的: 学会使用channel在协程间协调同步
func ProducerConsumer() {
	ch := make(chan int, 100)
	// 生产者
	waitGroup := sync.WaitGroup{}
	waitGroup.Add(2) // 2个生产者
	go func() {
		defer waitGroup.Done()
		for i := 0; i < 10; i++ {
			ch <- i
		}
	}()
	go func() {
		defer waitGroup.Done()
		for i := 0; i < 10; i++ {
			ch <- i
		}
	}()

	// 消费者
	// 因为blockChan是一个无缓冲的通道,所以会阻塞在这里
	// 设置类型为struct{}的原因是因为struct{}不占用内存空间
	blockChan := make(chan struct{}, 0)
	go func() {
		sum := 0
		for {
			a, ojbk := <-ch
			if !ojbk { // 通道关闭&&通道中没有数据 此时会返回false
				break
			}
			sum += a
		}
		fmt.Printf("sum=%d\n", sum)
		blockChan <- struct{}{}
	}()
	waitGroup.Wait()
	close(ch)
	// 会自动阻塞直到往blockChan中存放数据
	<-blockChan
}
```

#### Q3. 对go中的死锁的初次探究

```go
func DeadLock() {
	ch := make(chan int, 0)
	//go func() {
	//	ch <- 1
	//	fmt.Println("over")
	//}()

	//time.Sleep(time.Hour) 通过sleep会使得main协程一直阻塞
	// ch <- 2 //通过channel阻塞main 会被系统检测到死锁 从而报错 fatal error: all goroutines are asleep - deadlock!
	//wg := sync.WaitGroup{}
	//wg.Add(1)
	//go func() {
	//	defer wg.Done()
	//	ch <- 1
	//	fmt.Println("over")
	//}()
	//
	//wg.Wait() // 通过waitGroup阻塞main 会被系统检测到死锁 从而报错 fatal error: all goroutines are asleep - deadlock!

	wg := sync.WaitGroup{}
	wg.Add(1)
	go func() {
		defer wg.Done()
		ch <- 1
		fmt.Println("over")
	}()
	
	// 新加一个协程
	go func() {
		time.Sleep(time.Hour)
	}()

	wg.Wait() // 此时 系统不会检测到死锁 因为有一个协程在睡眠,系统无法判定在该协程睡眠过后是否会向ch中写入数据/将等待组减一 所以无法检测到死锁
}
```

死锁小demo:

![image-20230621174242261](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image-20230621174242261.png)

>这段代码创建了一个无缓冲的整型管道`ch`，并向管道中写入了一个整数1。由于管道是无缓冲的，因此写入操作会被阻塞，直到有其他goroutine从管道中读取数据。如果没有其他goroutine从管道中读取数据，那么写入操作将一直被阻塞，直到程序被终止或者有其他goroutine从管道中读取数据为止。
>
>由于管道是一种同步机制，因此在使用无缓冲管道时，写入操作和读取操作必须在不同的goroutine中进行，否则会导致死锁。如果在同一个goroutine中既进行写入操作又进行读取操作，那么程序将会被阻塞，无法继续执行。

### Gorm操作数据库

```go
// 文档地址 : https://gorm.io/zh_CN/docs/update.html
import (
	"fmt"
	"gorm.io/driver/mysql"
	"strconv"
)
import "gorm.io/gorm"

// User 用户表映射
type User struct {
	Id   int64  `json:"id"`
	Name string `gorm:"column:uname" json:"name"`
}

var client = connectMysql()

// TableName 重写表名
func (User) TableName() string {
	return "t_user"
}
func DBCURD() {
	//user := read()
	//if user == nil {
	//	fmt.Println("表中没有数据")
	//	return
	//}
	//fmt.Printf("读取user: %v\n", user)
	//
	//insert()
	//benchInsert()
	//readAll()
	//update()
	delete()
}
func delete() {

	result := client.Where("length(id) > ?", 2).Delete(&User{})
	fmt.Printf("删除的记录数是:%v", result.RowsAffected)
	err := result.Error
	handleError(err)
}

func update() {
	user := User{
		Id: 52222,
		//Name: "张宝磊",
	}
	//err := client.Save(&user).Error
	//handleError(err)
	//err := client.Model(&user).Update("uname", "张宝磊儿子").Error
	//handleError(err)
	err := client.Model(&user).Where("id = ?", 52222).Update("uname", "张堡垒孙子").Error
	handleError(err)
}

// 读取全部数据
func readAll() {
	var users []User
	result := client.Find(&users)
	fmt.Printf("总记录是:%v", result.RowsAffected)
	err := result.Error
	handleError(err)
	fmt.Printf("读取user: %v\n", users)
}

// 读取数据
func read() *User {
	var user User
	err := client.Take(&user).Error
	handleError(err)
	return &user
}

// 插入数据
func insert() {
	user := User{
		Id:   52222,
		Name: "coder4j",
	}
	err := client.Create(&user).Error
	handleError(err)
}

func benchInsert() {
	var users []User
	for i := 0; i < 1000; i++ {
		var user = User{
			Id:   int64(52223 + i),
			Name: "coder" + strconv.Itoa(i),
		}
		users = append(users, user)
	}
	err := client.CreateInBatches(&users, len(users)).Error
	handleError(err)
}

// 连接数据库
func connectMysql() *gorm.DB {
	dataSourceName := "root:coder4jdocker@tcp(112.126.71.240:3340)/db_user?charset=utf8mb4&parseTime=True&loc=Local"
	client, err := gorm.Open(
		mysql.Open(dataSourceName),
		nil,
	)
	handleError(err)
	return client
}

// 处理error
func handleError(err error) {
	if err != nil {
		panic(err)
	}
}
```



### Go操作Redis

```go

```



### Gin网络编程



### GRPC微服务

protoc  --go_out=plugins=grpc: . --proto_path=./idl -I=./idl/third_proto student_service.proto
