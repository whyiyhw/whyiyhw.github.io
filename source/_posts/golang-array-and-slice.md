---
title: golang base (one)
categories: golang
tags:
  - golang
abbrlink: fd1bbd3a
date: 2019-05-08 21:58:26
---

## 程序入口

- 必须为 `package main`
- 必须是 `func main(){}`
- 文件名称可以不为 `main.go`
- `Go` 中 `main` 函数不支持返回值
- 可以通过 `os.Exit()` 来传出 返回值
- `main` 函数不支持传入参数 可以通过 `os.Args` 来获取

  ```Go
    func main(){// 不支持入参
        fmt.Println("hello world!")
        if len(os.Args) >1 {//通过 os.Args 来获取传入的参数
            for arg,value := range os.Args{
                fmt.Println(arg,value)
            }
        }

        os.Exit(0)
    }
  ```

## 变量，常量以及与其它语言的区别

### 编写测试程序

- 源码文件以 `_test` 结尾： `xxx_test.go`
- 测试方法名以 `Test` 开头 `TestXXX(t *testing.T){...}`
- `go test -v xxx_test.go` 才能输出 `t.Log` 里的文字
- 实现斐波拉契数列

  ```Go
    func TestFibList(t *testing.T) {
        a,b := 1,1
        for i := 0; i < 5; i++ {
            fmt.Println(b)
            a,b = b,a+b
        }
    }
  ```

### 变量与其它静态编程语言的差异

- 赋值可以进行自动类型推断
- 同一个赋值语句中可以对多个变量进行同时赋值

### 常量与其它静态编程语言的差异

- 快速设置连续值 `iota` 遇到下一个 `const` 之前连续递增1,遇到之后变为0
- `iota` 只能在常量中使用
  
  ```Go
    const (
        Monday    = iota + 1 // 1
        Tuesday              // 2
        Wednesday            // 3
    )

    const (
        Readable   = 1 << iota //1 0001
        Writable               //2 0010
        Executable             //4 0100
    )

    const (
        i = iota // i=0
        j = 3.14 // j=3.14
        k = iota // k=2
        l        // l=3
    )

    type ByteSize float64

    const (
        _           = iota             // ignore first value by assigning to blank identifier
        KB ByteSize = 1 << (10 * iota) // 1 << (10*1)
        MB                             // 1 << (10*2)
        GB                             // 1 << (10*3)
        TB                             // 1 << (10*4)
        PB                             // 1 << (10*5)
        EB                             // 1 << (10*6)
        ZB                             // 1 << (10*7)
        YB                             // 1 << (10*8)
    )

  ```

## 数据类型

- 基本数据类型 值类型 初始化时默认会有零值
  - `bool` : `false`
  - `string` : `""`
  - `int int8 int16 int32 int64` : `0`
  - `uint uint8 uint16 uint32 uint64 uintptr` : `0`
  - `byte` : `0` // alias for uint8
  - `rune`: `0` // alias for int32，represents a Unicode code point
  - `float32 float64`:`0`
  - `complex64 complex128`: `(0+0i)`
- 复合类型
  - `pointer function interface slice channel map` : `nil`
  - 对于复合类型, `go` 语言会自动递归地将每一个元素初始化为其类型对应的零值。比如：数组， 结构体
  
### 整型占用字节问题

- int，uint整型：和机器平台有关，最小32位，占用4字节，64位，占用8字节。
  
  ```go
  //机器位数
  cpu := runtime.GOARCH
  t.Log(cpu) // amd64
  //int占用位数
  int_size := strconv.IntSize
  t.Log(int_size) // 64
  ```

### 数值范围

类型 | 长度（字节） | 数值范围
--- |--- |---
int8 | 1 | -128~127 （-2^(8-1) ~ 2^7-1）
uint8 | 1 | 0~255 (0 ~ 2^8-1)
int16 | 2 | -32768~32767
uint16 | 2 | 0~65535
int32 | 4 | -2^31 ~ 2^31-1 (-2147483648~2147483647)
uint32 | 4 | 0~2^32-1 (0~4294967295)
int64 | 8 | -2^63 ~2^63-1
uint64 | 8 | 0～2^63

### 数据类型与其它静态编程语言的差异

- Go语言不允许隐式类型转换
- 别名和原有类型也不能进行隐式类型转换
  - 在某些语言中允许小范围类型向大范围类型转换，因为数据精度不会丢失
  - 大范围向小范围转换会导致精度丢失
  
  ```Go
    type MyInt int64
    func TestDataType(t *testing.T){
        var a int32 = 1
        var b int64
        b = int64(a)
        var c MyInt
        c = MyInt(b)
        t.Log(a,b,c)
    }
  ```

- `Go` 语言不支持指针运算
- `string` 是值类型，其默认的初始值为 `""` 空字符串，而不是`nil`

  ```Go
  func TestPoint(t *testing.T) {
        a := 1
        aPtr := &a
        // aPtr = aPtr + 1 是不被允许的
        t.Log(a, aPtr)// 1 0xc00000a2b8
        t.Logf("%T %T", a, aPtr)// int *int
        var s string
        if s == "" {
            t.Log("string 零值为空字符串")
        }
        t.Log(len(s)) //0
    }
  ```

[golang代码编写最佳实践](https://mp.weixin.qq.com/s/BbZcp5OJSQHNi6nlnu3_eA)

## 运算符号与其它静态编程语言的差异

### 算术运算符

- `+ - * / % 后++ 后--`

### 比较运算符

- `==` `!=` `>` `<` `>=` `<=`
- 在 `golang` 比较数组 如果两个数组的维度相等是可以比较的
  
  ```Go
  a :=[...]int{1,2,3,4} // 数组声明
  b :=[...]int{1,2,4,3}
  c :=[...]int{1,2,3,4}
  t.Log(a == b) // false
  t.Log(a == c) // true
  ```

### 逻辑运算符

- `！`  `&&`  `||`
  
### 位运算符号

- `&` 按位与（A & B） 结果为12 二进制为00001100
- `|` 按位或（A|B）结果为61，二进制为00111101
- `^` 按位异或（A^B）结果为49，二进制为00110001
- `<<`A << 2 结果为240 ，二进制为11110000
- `>>`A >> 2 结果为15 ，二进制为00001111
- `&^` 按位清零 运算符 右边为1 则左边清零 为0则左边为原值

  ```Go
    1 &^ 0 --> 1 // 右边为0 则左边不变
    1 &^ 1 --> 0 // 右边为1 则左边清零
    0 &^ 1 --> 0 // 右边为1 则左边清零
    0 &^ 0 --> 1 // 右边为0 则左边不变
  ```
  
## 第八讲 流程控制

### 只有 `for` 一个关键字

- `for` 不需要前后的括号

```go
  // while 循环
  func TestWhileLoop(t *testing.T){
    n:= 0
    for n <5{
    n++
    t.Log(n)
    }
    // 无限循环
    for{

    }
    // 这个类似于 goto 与 php 的 break 2 跳出指定的控制结构
    n:= 0
    m:=0
    I:
    for m < 5 {
        m++
        t.Log(m)
        for n <5{
            n++
            if n>3{
                break I
            }
            t.Log(n)
        }
    }
  }
```

### `if` 条件

- 只有结果为 `bool` 值，才能成为 `if` 的条件

```go
if condition {
    // code will be executed if condition is true
}  else if condition-1 {
    // code will be executed if condition-1 is true
} else {
    // code will be executed if condition-1 and condition are false
}
// 前半部分支持 赋值
if var delaration; condition{

}
```

### `switch` 条件

- 条件表达式 不限制为常量或者整数
- 单个case中，可以出现多个结果选项，用逗号分隔
- 与 C 语言规则相反， Go 不需要加上break 来明确推出一个case
- 可以不设定 swtich 之后的条件表达式，在此种情况下，整个 switch 结构与多个 if...else... 的逻辑作用相同

  ```go
    for i := 0; i < 5; i++ {
        switch i {
        case 0, 2:
            t.Log("Even")
        case 1, 3:
            t.Log("Odd")
        default:
            t.Log("it is not 0-3")
        }
    }
    for i := 0; i < 5; i++ {
        switch  {
        case i%2 == 0:
            t.Log("Even")
        case i%2 != 0:
            t.Log("Odd")
        default:
            t.Log("unKnow")
        }
    }
  ```

## 第九讲 数组与切片

### 数组

- 数组的声明
  
```go
    var arr [3]int            // 声明并初始化为默认零值
    t.Log(arr[0], arr[1])     // 0,0
    arr1 := [3]int{1}         // 声明并初始化
    t.Log(arr1[0], arr1[1])   // 1,0
    arr2 := [...]int{1, 1, 1} // 声明并初始化 ... 代表根据后面的值来确定前面的长度
    t.Log(arr2[0], arr2[1])   // 1,1
    arr2[2] = 3               // 修改 直接访问下标即可

    // 多维数组的声明
    b := [2][3]int{{1, 2, 3}, {2, 3}}
    t.Log(b)
    c := [2][2]int{{2, 3}, {1, 2}}
    t.Log(c)
```

- 数组与 `slice` 的循环

```go
    arr := [...]int{1, 2, 3, 4, 5, 6}
    // 原始 for 循环
    for i := 0; i < len(arr); i++ {
        t.Log(arr[i])
    }
    // for range 类似于 php foreach
    for _, e := range arr {
        t.Log(e)
    }
```

- 数组与slice 的截取

```go
    // 数组截取操作 包括开始不包括结束
    arr := [...]int{1, 2, 3, 4, 5, 6}
    a1 := arr[:3]
    a2 := arr[1:3]
    a3 := arr[1:]
    a4 := arr[1:2]
    a4 = append(a4, 1)
    t.Log(len(a4), cap(a4))// len 2 cap 5
    t.Log(a4[1])
    // 截取出来的东西 实际上是一个 slice
    // 修改slice的值后 其它截取的slice 都发生了改变 也就是 说
    // slice 实际上是对于 arr 的一个引用 并没有实际的空间去存储值
    t.Log(a1, a2, a3, a4, arr)
```

### 切片

```go
    // slice 可以看作一个结构体 包括三部分
    // 指针 指向连续内存 数组 *ptr
    // len 元素的个数 len
    // cap 内部数组的容量

    // 声明 区别在于 不用加 ... 与显示的申明长度
    var s0 []int
    s0 = append(s0, 1) // 增 append 后记得接受值
    t.Log(len(s0), cap(s0))//1,1

    s1 := []int{1, 2}
    t.Log(len(s1), cap(s1))//2,2

    // make 可用于声明一个 slice 第二个c
    s2 := make([]int, 3, 5) // 长度为3 容量为5
    t.Log(len(s2), cap(s2))
    t.Log(s2[0], s2[1], s2[2])
    // len 初始化元素的个数
    // cap 总容量
    s2 = append(s2, 1)
    t.Log(len(s2), cap(s2)) // 4,5
    s2 = append(s2, 2)
    t.Log(len(s2), cap(s2)) // 5,5
    s2 = append(s2, 3)
    t.Log(len(s2), cap(s2)) // 6,10
    // 如果新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）
    // 如果旧切片长度大于等于1024，则最终容量（newcap）从旧容量（old.cap）开始循环增加原来的 1/4，即（newcap=old.cap,for {newcap += newcap/4}）直到最终容量（newcap）大于等于新申请的容量(cap)，即（newcap >= cap）
    // 如果最终容量（cap）计算值溢出，则最终容量（cap）就是新申请容量(cap)

    // 切片共享存储结构
    year := []string{
        "Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct",
        "Nov", "Dec"}
    Q2 := year[3:6]
    t.Log(Q2, len(Q2), cap(Q2))// 3,9
    summer := year[5:8]
    t.Log(summer,len(summer), cap(summer))
    summer[0] = "unKnow"
    t.Log(Q2,year)
    // 共享内存修改值会相互影响
```

#### 区别

- 容量是否可伸缩
  - 切片容量可伸缩
    - 首先判断，如果新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）
    - 否则判断，如果旧切片的长度小于1024，则最终容量(newcap)就是旧容量(old.cap)的两倍，即（newcap=doublecap）
    - 否则判断，如果旧切片长度大于等于1024，则最终容量（newcap）从旧容量（old.cap）开始循环增加原来的 1/4，即（newcap=old.cap,for {newcap += newcap/4}）直到最终容量（newcap）大于等于新申请的容量(cap)，即（newcap >= cap）
    - 如果最终容量（cap）计算值溢出，则最终容量（cap）就是新申请容量(cap)
    - slice的共享内存存在一个隐患，就是使用共享内存的双方slice如果有一方出现扩容，则扩容一方将不再指向原有空间。所以程序中想以修改一方达到数据同步时需要小心
    - 扩容后会指向一个更大的空间,不再指向原有空间,所以共享内存失效
  - 数组声明后长度不可修改
  
- 是否可比较
  - 数组可以比较 同一类型 同一长度的数组可以比较
  - slice 只能与 nil 进行比较
