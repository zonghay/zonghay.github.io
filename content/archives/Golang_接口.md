---
title: "Golang 接口(Interface)"
date: 2025-02-06T17:36:57+08:00
lastmod: 2025-02-06T17:36:57+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- Golang
description: ""
weight:
slug: ""
draft: false # 是否为草稿
comments: true # 本页面是否显示评论
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
hideSummary: false
ShowWordCount: true
ShowBreadCrumbs: true #顶部显示路径
cover:
    image: "" #图片路径例如：posts/tech/123/123.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

一则简单的例子

```go
type IGreeting interface {
	sayHello()
}

func sayHello(i IGreeting) {
	i.sayHello()
}

type Go struct {}
func (g Go) sayHello() {
    fmt.Println("Hi, I am GO!")
}

type PHP struct {}
func (p PHP) sayHello() {
    fmt.Println("Hi, I am PHP!")
}

func main() {
    golang := Go{}
    php := PHP{}

    sayHello(golang)
    sayHello(php)
}
```

## 如何通过接口实现多态
多态的字面意思是“多种形态”。在编程中，它指的是：
* 同一操作作用于不同的对象，可以有不同的解释，产生不同的执行结果。
* 通过多态，程序可以在运行时根据对象的实际类型来决定调用哪个方法，而不是在编译时固定。

```go
package main

import "fmt"

// 定义接口
type Animal interface {
	Speak() string
}

// 实现接口的类型
type Dog struct{}

func (d Dog) Speak() string {
	return "Woof!"
}

type Cat struct{}

func (c Cat) Speak() string {
	return "Meow!"
}

type Cow struct{}

func (c Cow) Speak() string {
	return "Moo!"
}

func main() {
	// 多态：通过接口调用具体类型的方法
	animals := []Animal{Dog{}, Cat{}, Cow{}}
	for _, animal := range animals {
		fmt.Println(animal.Speak())
	}
}
```
在这个示例中：
* Dog、Cat 和 Cow 都实现了 Animal 接口。
* 通过 Animal 接口调用 Speak 方法时，程序会根据具体的类型（Dog、Cat 或 Cow）执行对应的方法。

## Golang和PHP接口的异同
### 接口定义
#### Go
- Go 的接口是隐式实现的，不需要显式声明某个类型实现了某个接口。
- 接口定义了一组方法签名，任何类型只要实现了这些方法，就自动实现了该接口。

#### PHP
- PHP 的接口是显式实现的，需要使用 `implements` 关键字声明某个类实现了某个接口。
- 接口定义了一组方法签名，类必须显式实现接口中定义的所有方法。

### 接口组合
#### Go 
- Go 支持接口的组合，可以通过组合多个接口来定义新的接口。
  ```go
  type Reader interface {
      Read([]byte) (int, error)
  }

  type Writer interface {
      Write([]byte) (int, error)
  }

  type ReadWriter interface {
      Reader
      Writer
  }
  ```
  这里 `ReadWriter` 接口组合了 `Reader` 和 `Writer` 接口。

#### PHP
- PHP 中的类可以通过`implements`关键词组合实现多接口
```php
interface InterfaceA {
    public function methodA();
}

interface InterfaceB {
    public function methodB();
}

class MyClass implements InterfaceA, InterfaceB {
    public function methodA() {
        echo "Method A implemented.";
    }

    public function methodB() {
        echo "Method B implemented.";
    }
}
```

### 总结

| 特性                | Go 语言                          | PHP                |
|---------------------|----------------------------------|--------------------|
| 实现方式            | 隐式实现                         | 显式实现（`implements`） |
| 灵活性              | 更高（无需修改类型定义）         | 较低（需修改类定义）         |
| 接口组合            | 支持                             | 支持（类implement多接口）  |
| 空接口              | 支持（`interface{}`）            | 不支持 |
| 类型系统            | 静态类型                         | 动态类型               |

## 函数类型也能实现接口
乍一听是不是感觉很神奇？！       
来看标准库net/http中的经典代码
```go
package http

type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

func Handle(pattern string, handler Handler) {
  ...
}

type HandlerFunc func(ResponseWriter, *Request)

// 为 HandlerFunc 实现 ServeHTTP 方法
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```
使用方式：
```go
func helloHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello, World!"))
}

func main() {
    http.Handle("/", http.HandlerFunc(helloHandler))
    http.ListenAndServe(":8080", nil)
}
```
> 思考一个问题：为什么 http.Handle 函数的第二个参数不直接声明为 HandlerFunc，而是声明为 Handler 接口呢？

## 编译器自动检测类型是否实现接口
经常看到一些开源库里会有一些类似下面这种奇怪的用法：
> ```var _ io.Writer = (*myWriter)(nil)```

这时候会有点懵，不知道作者想要干什么，实际上这就是此问题的答案。编译器会由此检查 `*myWriter` 类型是否实现了 `io.Writer` 接口。

来看一个例子：
```go
package main

import "io"

type myWriter struct {

}

/*func (w myWriter) Write(p []byte) (n int, err error) {
	return
}*/

func main() {
    // 检查 *myWriter 类型是否实现了 io.Writer 接口
    var _ io.Writer = (*myWriter)(nil)

    // 检查 myWriter 类型是否实现了 io.Writer 接口
    var _ io.Writer = myWriter{}
}
```
注释掉为 myWriter 定义的 Write 函数后，运行程序：
```
src/main.go:14:6: cannot use (*myWriter)(nil) (type *myWriter) as type io.Writer in assignment:
	*myWriter does not implement io.Writer (missing Write method)
src/main.go:15:6: cannot use myWriter literal (type myWriter) as type io.Writer in assignment:
	myWriter does not implement io.Writer (missing Write method)
```
报错信息：`*myWriter/myWriter` 未实现 `io.Writer` 接口，也就是未实现 Write 方法。

解除注释后，运行程序不报错。

实际上，上述赋值语句会发生隐式地类型转换，在转换的过程中，编译器会检测等号右边的类型是否实现了等号左边接口所规定的函数。

## 值接收者和指针接收者

### 普通方法
直接讲结论：对于普通方法(不实现接口的方法)调用，不管方法的接收者是什么类型，该类型的值和指针都可以调用，不必严格符合接收者的类型。

```go
package main

import "fmt"

type Person struct {
	age int
}

func (p Person) howOld() int {
	return p.age
}

func (p *Person) growUp() {
	p.age += 1
}

func main() {
	// qcrao 是值类型
	qcrao := Person{age: 18}

	// 值类型 调用接收者也是值类型的方法
	fmt.Println(qcrao.howOld())

	// 值类型 调用接收者是指针类型的方法
	qcrao.growUp()
	fmt.Println(qcrao.howOld())

	// ----------------------

	// stefno 是指针类型
	stefno := &Person{age: 100}

	// 指针类型 调用接收者是值类型的方法
	fmt.Println(stefno.howOld())

	// 指针类型 调用接收者也是指针类型的方法
	stefno.growUp()
	fmt.Println(stefno.howOld())
}

/*
OUTPUT:
  18
  19
  100
  101
*/
```

实际上，当类型和方法的接收者类型不同时，其实是编译器在背后做了一些工作，用一个表格来呈现：

|         | **值接收者**     | **指针接收者**     |
|---------| -------- | -------- |
| 值类型调用者  | 方法会使用调用者的一个副本，类似于“传值”     | 使用值的引用来调用方法，上例中，`qcrao.growUp()` 实际上是 `(&qcrao).growUp()`     |
| 指针类型调用者 | 指针被解引用为值，上例中，`stefno.howOld()` 实际上是` (*stefno).howOld() `    | 实际上也是“传值”，方法里的操作会影响到调用者，类似于指针传参，拷贝了一份指针     |


### 实现接口的方法
而对于实现某个接口的方法来讲，情况就有点不一样了。       

再来看个例子：
```go
type MyInterface interface {
    PointerMethod()
}

type MyStruct struct {
    value int
}

// 指针接收者的方法
func (m *MyStruct) PointerMethod() {
    fmt.Println("Pointer method called")
}

func main() {
    var i MyInterface
    m := MyStruct{} // 值类型的对象

    i = m // 错误：MyStruct 没有实现 MyInterface
    i = &m // 正确：*MyStruct 实现了 MyInterface
}
```
当涉及到接口实现时，接口的实现规则要求：
* 如果方法的接收者是指针类型(*T)，那么只有指针类型的对象(*T)才能实现该接口。
* 值类型的对象(T)无法实现包含指针接收者方法的接口。

上面的说法有一个简单的解释：接收者是指针类型的方法，很可能在方法中会对接收者的属性进行更改操作，从而影响接收者；而对于接收者是值类型的方法，在方法中不会对接收者本身产生影响。         
所以，当实现了一个接收者是值类型的方法，就可以自动生成一个接收者是对应指针类型的方法，因为两者都不会影响接收者。但是，当实现了一个接收者是指针类型的方法，如果此时自动生成一个接收者是值类型的方法，原本期望对接收者的改变（通过指针实现），现在无法实现，因为值类型会产生一个拷贝，不会真正影响调用者。

### 使用场景

1. 如果类型的本质是“原始的”（primitive），或者其成员是内置的引用类型（如 slice、map、interface、channel），则使用值接收者。因为这些类型的复制是安全的，且成本较低。
```go
type Point struct {
    X, Y int
}

func (p Point) Distance() float64 {
    return math.Sqrt(float64(p.X*p.X + p.Y*p.Y))
}
```
2. 如果类型的本质是“非原始的”，或者不能被安全地复制（例如文件句柄、数据库连接等），则使用指针接收者。因为这些类型的复制会导致资源管理问题。
```go
type DBConnection struct {
    conn *sql.DB
}

func (db *DBConnection) Query(query string) (*sql.Rows, error) {
    return db.conn.Query(query)
}
```
3. 是否修改接收者
```go
type Counter struct {
    count int
}

func (c *Counter) Increment() {
    c.count++
}
```
这里 `Increment` 方法使用指针接收者，修改的是 c 本身。

## 接口类型和 nil 作比较
接口值的零值是指 **动态类型** 和 **动态值** 都为 nil。当仅且当这两部分的值都为 nil 的情况下，这个接口值就才会被认为 **接口值 == nil**。

```go
package main

import "fmt"

type Coder interface {
	code()
}

type Gopher struct {
	name string
}

func (g Gopher) code() {
	fmt.Printf("%s is coding\n", g.name)
}

func main() {
	var c Coder
	fmt.Println(c == nil)
	fmt.Printf("c: %T, %v\n", c, c)

	var g *Gopher
	fmt.Println(g == nil)

	c = g
	fmt.Println(c == nil)
	fmt.Printf("c: %T, %v\n", c, c)
}

// OUTPUT:
/*true
c: <nil>, <nil>
true
false
c: *main.Gopher, <nil>*/
```
一开始，`c` 的动态类型和动态值都为 `nil`，`g` 也为 `nil`，当把 `g` 赋值给 `c` 后，`c` 的动态类型变成了 `*main.Gopher`，仅管 `c` 的动态值仍为 `nil`，但是当 `c` 和 `nil` 作比较的时候，结果就是 `false` 了。

```go
package main

import "fmt"

type MyError struct {}

func (i MyError) Error() string {
	return "MyError"
}

func main() {
	err := Process()
	fmt.Println(err)

	fmt.Println(err == nil)
}

func Process() error {
	var err *MyError = nil
	return err
}

// OUTPUT
/*<nil>
false*/
```
这里先定义了一个 `MyError` 结构体，实现了 `Error` 函数，也就实现了 `error` 接口。`Process` 函数返回了一个 `error` 接口，这块隐含了类型转换。所以，虽然它的值是 `nil`，其实它的类型是 `MyError`，最后和 `nil` 比较的时候，结果为 `false`。

## 类型转换和断言的区别
我们知道，Go 语言中不允许隐式类型转换，也就是说 `=` 两边，不允许出现类型不相同的变量。

**类型转换** 、**类型断言** 本质都是把一个类型转换成另外一个类型。不同之处在于，类型断言是对接口变量进行的操作。

### 类型转换
对于 **类型转换** 而言，转换前后的两个类型要相互兼容才行。类型转换的语法为：
> <结果类型> := <目标类型> ( <表达式> )

```go
package main

import "fmt"

func main() {
	var i int = 9

	var f float64
	f = float64(i)
	fmt.Printf("%T, %v\n", f, f)

	f = 10.8
	a := int(f)
	fmt.Printf("%T, %v\n", a, a)

	// s := []int(i)
}
```
上面的代码里，我定义了一个 int 型和 float64 型的变量，尝试在它们之前相互转换，结果是成功的：int 型和 float64 是相互兼容的。

如果我把最后一行代码的注释去掉，编译器会报告类型不兼容的错误：     
```cannot convert i (type int) to type []int```

### 断言
因为空接口 interface{} 没有定义任何函数，因此 Go 中所有类型都实现了空接口。当一个函数的形参是 interface{}，那么在函数中，需要对形参进行断言，从而得到它的真实类型。

断言的语法为：     
> <目标类型的值>，<布尔参数> := <表达式>.( 目标类型 ) // 安全类型断言
> 
> <目标类型的值> := <表达式>.( 目标类型 )　　//非安全类型断言

类型转换和类型断言有些相似，不同之处，在于类型断言是对接口进行的操作。
```go
package main

import "fmt"

type Student struct {
	Name string
	Age int
}

func main() {
	var i interface{} = new(Student)
	s := i.(Student)
	
	fmt.Println(s)
}
```
运行结果：
```panic: interface conversion: interface {} is *main.Student, not main.Student```
直接 panic 了，这是因为 i 是 *Student 类型，并非 Student 类型，断言失败。

> `fmt.Println` 函数的参数是 `interface`。对于内置类型，函数内部会用穷举法，得出它的真实类型，然后转换为字符串打印。而对于自定义类型，首先确定该类型是否实现了 `String()` 方法，如果实现了，则直接打印输出 `String()` 方法的结果；否则，会通过反射来遍历对象的成员进行打印。

## 参考
[https://golang.design/go-questions/interface/duck-typing/](https://golang.design/go-questions/interface/duck-typing/)
