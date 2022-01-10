---
title: go design (四) string
date: 2022-01-08 11:57:55
categories: golang 
tags:
- string
- 《go语言设计与实现》
---

# golang 中 字符串 的设计

字符串是由字符组成的数组，C 语言中的字符串使用字符数组 `char[]` 表示。数组会占用一片连续的内存空间，而内存空间存储的字节共同组成了字符串，`Go` 语言中的字符串为一个只读的字节数组。

## `golang` 中对于 字符串 的设计

### string 数据结构 

```go
// reflect/value.go 
type StringHeader struct {
	Data uintptr
	Len  int
}

// slice 的定义
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

- 根据源码来看 其实 `string` 跟 `slice` 除了 少了 一个 `cap` 属性，其他的都一致,所以我们经常会说字符串是一个只读的切片类型
- 字符串作为只读的类型，我们无法直接向字符串直接追加元素改变其本身的内存空间，所有在字符串上的写入操作都是通过拷贝实现的。

### 修改，拼接与类型转换

#### string 的修改

```go
str := "string"
fmt.Println(reflect.TypeOf(str[0]).Name())// uint8 
//也就是说 通过 slice 方式去读取的是每一字节的数据
// 对于 中文获取 长字节 的字符，不能直接通过这种方式去获取

// 会触发编译错误 string 是只读不变的
// str[0] = '0'  cannot assign to str[0] (strings are immutable) 

s1 := []byte(str)
s1[1] = 'o'
fmt.Println(str,string(s1))// string soring
```

- 修改只能通过 `[]byte` 进行类型转换后，才能对 `slice` 进行修改，从 结果来看 ，类型转换是直接 `copy` 来处理的，这也会对性能造成影响

#### string 的拼接

```go
str := "string"
// 方式一
str = str + "233"
// 方式二
str = strings.Join([]string{str, "233"}, "")
// 方式三
str = fmt.Sprintf("%s%s", str, "233")
// 方式四
var build strings.Builder
build.WriteString(str)
build.WriteString("233")
str = build.String()
// 方式五
var bt bytes.Buffer
bt.WriteString(str)
bt.WriteString("233")
str = bt.String()
```

- `+` 连接适用于短小的、常量字符串（明确的，非变量），编译器会进行优化，综合最为推荐的为 `strings.builder `

#### string 的类型转换

- `string` 主要 转换为 `[]byte` 则会调用底层的 `stringtoslicebyte()` 方法

```go
//runtime/string.go
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) (str string) {
	if n == 0 {
		// Turns out to be a relatively common case.
		// Consider that you want to parse out data between parens in "foo()bar",
		// you find the indices and convert the subslice to string.
		return ""
	}
	if raceenabled {
		racereadrangepc(unsafe.Pointer(ptr),
			uintptr(n),
			getcallerpc(),
			funcPC(slicebytetostring))
	}
	if msanenabled {
		msanread(unsafe.Pointer(ptr), uintptr(n))
	}
	if n == 1 {
		p := unsafe.Pointer(&staticuint64s[*ptr])
		if sys.BigEndian {
			p = add(p, 7)
		}
		stringStructOf(&str).str = p
		stringStructOf(&str).len = 1
		return
	}

	var p unsafe.Pointer
	if buf != nil && n <= len(buf) {
		p = unsafe.Pointer(buf)
	} else {
		p = mallocgc(uintptr(n), nil, false)
	}
	stringStructOf(&str).str = p
	stringStructOf(&str).len = n
	memmove(p, unsafe.Pointer(ptr), uintptr(n))
	return
}


func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
	}
	copy(b, s)
	return b
}
```

- 拼接和类型转换等操作时会存在性能的损耗，遇到需要极致性能的场景要减少类型转换的次数

### 相关的小问题

#### `rune` 和 `byte` 有什么区别？

- `rune`  是 `int32` 的别名  4字节， `byte` 是 `int8` 的别名  1字节
- 所以 `byte`  只能 表示 `ascii` 的字符 `rune`  可以 表示 `Unicode` 字符