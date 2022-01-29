---
title: go design (一) array
date: 2021-12-31 20:24:03
categories: golang 
tags:
- array
- 《go语言设计与实现》
---

# golang 中 array 的实现

数组是由相同类型元素的集合组成的数据结构。
计算机操作系统会为数组分配一块连续的内存来保存其中的元素，我们可以利用数组中元素的索引快速访问特定元素 。
常见的数组大多都是一维的线性数组。多维数组在数值计算和图形应用方面非常有用。

## `golang` 中 对于数组的实现 `array`

```go
// cmd/compile/internal/types/type.go

// Array contains Type fields specific to array types.
type Array struct {
	Elem  *Type // element type
	Bound int64 // number of elements; <0 if unknown yet
}


// NewArray returns a new fixed-length array Type.
func NewArray(elem *Type, bound int64) *Type {
	if bound < 0 {
		base.Fatalf("NewArray: invalid bound %v", bound)
	}
	t := New(TARRAY)
	t.Extra = &Array{Elem: elem, Bound: bound}
	t.SetNotInHeap(elem.NotInHeap())
	if elem.HasTParam() {
		t.SetHasTParam(true)
	}
	return t
}

// A Type represents a Go type.
type Type struct {
    ······
    // Extra contains extra etype-specific fields.
	// TARRAY: *Array
	Extra interface{}

	// Width is the width of this Type in bytes.
	Width int64 // valid if Align > 0
    ......
}
```

- 由源码可以了解编译前的 `golang` 中的 `array` 由`elem`(任意类型元素 `Type`） 跟 `bound` 元素个数 组成, `type.Width` 代表了此元素的长度。知道数组的元素个数，元素的起始位置，每个元素的长度之后，后续的对于数组的操作就简单了。
- 对于元素类型一致但声明 `bound` 不一致 `array` 是不可比较的

![11](/img/11.png)

## array 的初始化

```go
	arr1 := [3]int{1, 2, 3}
	arr2 := [...]int{1, 2, 3}
	var arr3 = [3]int{1, 2, 3}
```

- 对于 [...] 声明的数组，底层会自动推导 `bound` 并转成 上一种声明方式


## 访问与赋值

- 数组的长度固定，那么对于简单的越界操作，编译时就能发现

```go
    arr2 := [...]int{1, 2, 3}
    arr2[3] = 2
    //invalid array index 3 (out of bounds for 3-element array)
    //Compilation finished with exit code 2
```

- 而对于变量去替代索引的情况，就只能由运行时去判定

```go
    arr2 := [...]int{1, 2, 3}
    rand.Seed(time.Now().Unix())
    i := rand.Int()
    arr2[i] = 2
    //panic: runtime error: index out of range [6089373529803393694] with length 3
```

- 如何证明array在内存中的连续性？

```go
    arr2 := [...]int{1, 2, 3, 4}
    fmt.Println(unsafe.Pointer(&arr2[0]), unsafe.Pointer(&arr2[1]), unsafe.Pointer(&arr2[2]), unsafe.Pointer(&arr2[3]))
    //0xc000010180 0xc000010188 0xc000010190 0xc000010198
```
  - 很显然 数组中每一位的地址 相差为 10 进制的 8 ，那就是说在我64位的机器上 int  为 8字节


  数组是程序中最基础的结构之一，知道索引时，其 `O(1)` 的读写速度也十分优秀，但其定长的特性，注定了灵活性的不足。`golang` 中 我们使用最多的 `slice` ，但其底层也是依赖于`array`。

  下一篇, 我们一起看看 `slice` 的设计与实现。

- 相关参考
	- [维基百科]：https://zh.wikipedia.org/wiki/%E6%95%B0%E7%BB%84
