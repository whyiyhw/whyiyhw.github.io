---
title: golang base (two)
date: 2019-06-08 22:37:49
categories: golang
tags: 
- golang
---
## map

- 声明
  
```go
    // 直接声明并赋值
    m := map[string]int{"one":1,"two":2,"three":3}
    t.Log(m, len(m))//map[one:1 two:2 three:3] len 3
    // 声明并赋予零值
    m1 := map[string]int{}
    m1["one"] = 1
    t.Log(m1, len(m1))//map[one:1] len 1
    // slice map
    m2 :=make(map[string]int,5/*cap 实际上预先设定了5的容量*/)// map 类型 ,len
    t.Log(m2,len(m2))//map[] len 0
```

- 访问不存在的key

```go
    m3 := map[int]int{}
    t.Log(m3[1]) // 0
    m3[2] = 0
    t.Log(m3[2]) // 0
    m3[3] = 0;
    // 判断存在或者为 零值
    if v, ok := m3[3]; ok {
        t.Log(v, "key 3 is exist.")
    } else {
        t.Log("KEY 3 is not existing.")
    }
    // 在访问 不存在的 key 的时候 会返回对应的零值 而不是异常
```

- 遍历
  
```go
    m := map[string]int{"one": 1, "two": 2, "three": 3}
    for k, v := range m {
        t.Log(k, "=>", v)
    }
```

- map 实现工厂模式

```go
    m := map[int]func(op int) int{}//
    m[1] = func(op int) int {
        return op
    }
    m[2] = func(op int) int {
        return op * op
    }
    m[3] = func(op int) int {
        return op *op* op
    }
    t.Log(m[1](2),m[2](2),m[3](2))// 2 4 8
```

- 用 `map` 实现 `set` 的功能

```go
// 元素是唯一的
// 1) 添加元素
// 2) 判断元素是否存在
// 3) 删除元素
// 4) 元素个数
    mySet := map[int]bool{}
    mySet[1] = true
    mySet[3] = true
    n:= 3
    if mySet[n]{
        t.Logf("%d is existing",n)
    }else {
        t.Logf("%d is not existing",n)
        t.Log(mySet[n])
    }

    t.Log(len(mySet))
    delete(mySet, n)

    if mySet[n]{
        t.Logf("%d is existing",n)
    }else {
        t.Logf("%d is not existing",n)
        t.Log(mySet[n])
    }
```

## 字符串

- `string` 是数据类型,不是引用或是指针类型,零值为空字符串
- `string` 是 只读的 `byte` `slice` , `len` 函数可以展示它所包含的 `byte`
- `string` 的 `byte` 数组可以存放任何数据

```go
    var s string
    t.Log(s)// ""
    s = "hello"
    // s[1] = '2'//can't assign s[1]
    t.Log(len(s))// 5
    s = "\xE4\xB8\xA5"
    t.Log(s,len(s))// 严 3 len 是 byte 长度
    // Unicode 是一种字符集（code point) 只规定了对应关系，没有规定怎么实现
    // UTF8 是 unicode 的存储实现（转换为字节序列的规则）

    s ="中"
    t.Log(len(s))// byte 数 3

    c:=[]rune(s)
    t.Log(len(c))// 1
    t.Log("rune size :", unsafe.Sizeof(c))// 24 3*8 就是 3 byte 24字节

    t.Logf("中 unicode %x",c[0])
    t.Logf("中 UTF8 %x",s)

    // 遍历string的时候，实际自动把string转成了rune再遍历。
    // 而不是根据 len(string)的byte数组个数
    s = "中华人民共和国"
    for _,v := range s {
        t.Logf("%[1]c %[1]x",v)
    }

    // strings strconv
    // 字符串分割 string
    s = "A,B,C"
    parts := strings.Split(s,",")// []string 的切片
    t.Logf("%T", parts)
    for _,v := range parts{
        t.Logf("%T", v)// string
    }
    // 合并
    t.Log(strings.Join(parts,"-"))
    // 转换
    s1 := strconv.Itoa(10)// Itoa AtoI
    t.Log("str "+ s1)
    if s2,err := strconv.Atoi(s1);err == nil{
        t.Log(10+s2)
    }
```

## 函数

- 函数是一等公民
  - 函数有多个返回值
  - 所有参数都是值传递：`slice` `map` `channel` 会有传引用的错觉
  - 函数可以作为变量的值
  - 函数可以作为参数和返回值

```go
// 支持多返回值
func returnMultiValues() (a, b int) {
    return rand.Intn(10), rand.Intn(20)
}
// 可以作为参数与返回值
func timeSpent(inner func(op int) int) (func(op int) int){
    return func(n int) int {
        start := time.Now()
        ret := inner(n)
        fmt.Println("time Spent:",time.Since(start).Seconds())
        return ret
    }
}
func slowFun(op int)int{
    time.Sleep(time.Second*1)
    return  op
}
func TestFn(t *testing.T) {
    i, l := returnMultiValues()
    t.Log(i, l)

    tsSF := timeSpent(slowFun)
    t.Log(tsSF(10))
}
```

### 可变长参数与 `defer`

```go
// 可变长参数 参数类型一致
func Sum(ops ...int)(res int){
    res = 0
    for _,v := range ops{
        res += v
    }
    return
}
func TestFn1(t *testing.T){
    t.Log(Sum(1,2,3,4,5))
    t.Log(Sum(1,2,3,4,5,6))
}


func Clear(){
    defer fmt.Println("errs")
    fmt.Println("close")
    panic("err")
}
func Clear2(){
    defer fmt.Println("errs2")
    fmt.Println("close2")
    panic("err")
}

func TestDefer(t *testing.T){
    defer Clear()
    defer Clear2()
    fmt.Println("Start")
    panic("err")
}
// 执行顺序为
// Start close2 errs2 close errs
// defer 类似于栈结构 先进后出
// 在 defer 中引发 panic 只会导致当前的 defer 体里面的程序中断
```

## 面向对象

### 封装数据和行为（方法）

```go
// 数据封装
type Employee struct {
    Id   string
    Name string
    Age  int
}

func TestCreateEmployeeObj(t *testing.T){
    e:= Employee{"0","张三",23}
    e1 := Employee{Id:"张三",Name:"李四",Age:25}
    e2 := new(Employee) // 返回的是一个指针
    e2.Age = 29
    e2.Name = "王五"
    e2.Id = "3"
    t.Log(e)
    t.Log(e1)
    t.Log(e2)
    t.Logf("e is %T", e)
    t.Logf("e2 is %T", e2)
}

//    main_test.go:437: {0 张三 23}
//    main_test.go:438: {张三 李四 25}
//    main_test.go:439: &{3 王五 29}
//    main_test.go:440: e is main.Employee
//    main_test.go:441: e2 is *main.Employee
```

```go
// 行为封装
// 通过引用去使用这个方法
func (e *Employee) String() string {
    fmt.Printf("Address is %x\n", unsafe.Pointer(&e.Name))
    return fmt.Sprintf("ID:%s-Name:%s-Age:%d", e.Id, e.Name, e.Age)
}
// 通过值拷贝去使用这个方法 这种会带来更大的内存消耗
func (e Employee) String() string {
    fmt.Printf("Address is %x\n", unsafe.Pointer(&e.Name))
    return fmt.Sprintf("ID:%s-Name:%s-Age:%d", e.Id, e.Name, e.Age)
}
```

### 接口的定义

- 非入侵的，实现不依赖接口定义
- 所以接口的定义可以包含在接口使用者包内
  
```go
type Programmer interface {
    WriterHelloWorld() string
}

type  GoProgrammer struct {

}
func (to *GoProgrammer) WriterHelloWorld() string{
    return "fmt.Println(\"hello world!\")"
}

func TestClient(t *testing.T){
    var p Programmer
    p = new(GoProgrammer)
    t.Log(p.WriterHelloWorld())
}

```

### 接口变量

- 声明 `var prog Coder = &GoPaogrammer{}`
- prog 由两部分组成
  - `type GoPaogrammer struct{}` 数据类型
  - `&GoPaogrammer{}` 数据实例
  
### 继承与组合

- golang 中组合的成分高于继承,甚至可以说没有一般语言的继承概念

```go
type Pet struct {

}

func (p *Pet) Speck()  {
    fmt.Println("...")
}

func (p *Pet) SpeckTo(host string) {
    p.Speck()
    fmt.Println(" ", host)
}

type Dog struct {
    //p *Pet // 复合
    Pet
}

func (d *Dog) Speck()  {
    fmt.Println("www")
    d.SpeckTo("laowan")
}

//func (d *Dog) SpeckTo(host string) {
//  //d.SpeckTo(host)
//  d.Speck()
//  fmt.Println(host)
//}

func TestInherit(t *testing.T){
    // var dog Pet = new(Dog) 不能使用 Dog 去作为 Pat 类型
    var dog = new(Dog)
    //dog.SpeckTo("张三") // ... 张三 匿名变量 只能调用 变量自己的方法
    dog.Speck() //www ... lanwan
    // 子类可以调用 匿名变量的方法但是这不算是完整意义上的继承

    // 关于继承这部分 跟 php 差距很大，与其说是继承，我更加倾向于说这是 组合
    // 类似于 php 中的 traits 但又不像，因为匿名变量分界线很清楚，
    // traits 是可以在被使用后 根据使用者的情况展示不同的特性
    // 是你的就是你的，是我的就是我的，你用我的，那就是我的，上层不能使用下层的方法
    // 下层在使用上层的方法时候，可以理解为全在上层使用
```

### 接口与多态

```go
type Code string
type Programmer interface {
    WriterHelloWorld() Code
}

type GoProgrammer struct {
}

func (to *GoProgrammer) WriterHelloWorld() Code {
    return "fmt.Println(\"hello world!\")"
}

type JavaProgrammer struct {
}

func (to *JavaProgrammer) WriterHelloWorld() Code {
    return "fmt.Println(\"hello world!\")"
}

func WriterFirstProgrammer(programmer Programmer) {
    fmt.Printf("%T %v\n", programmer, programmer.WriterHelloWorld())
}

func TestPolymorphism(t *testing.T) {
    goProg := new(GoProgrammer)// 指针
    javaProg := new(JavaProgrammer)// 指针
    WriterFirstProgrammer(goProg)
    WriterFirstProgrammer(javaProg)
}
```

### 空接口与断言

- 空接口可以表示任何类型
- 通过断言来将空接口转换为指定类型
  - v,ok := p.(int) // ok=true 时为转换成功
  
  ```go
  func DoSomething(p interface{}) {
    //if i,ok := p.(int);ok{
    // fmt.Println("Integer", i)
    // return
    //}
    //if i,ok := p.(string);ok{
    // fmt.Println("String", i)
    // return
    //}
    switch i := p.(type) {
    case int:
        fmt.Println("Integer", i)
        return
    case string:
        fmt.Println("String", i)
        return
    default:
        fmt.Println("unKnow Type")
    }
  }

  func TestDoSomething(t *testing.T) {
    DoSomething(10)
    DoSomething("10")
    DoSomething('1')
  }
  ```

### Go接口最佳实践

- 倾向于使用小的接口定义，很多接口只包含一个方法（对于实现者负担更新 single Interface）
  
  ```go
  type Reader interface{
      Read(p []byte)(n int, err error)
  }  
  type Writer interface{
      Write(p []byte)(n int, err error)
  }  
  ```

- 较大的接口定义，可以由多个小接口定义组合而成
  
  ```go
  type ReadWriter interface{
      Reader
      Writer
  }
  ```

- 只依赖必要功能的最小接口(会使方法的复用性更强)
  
  ```go
  func StoreData(reader Reader) error{
      ...
  }
  ```
