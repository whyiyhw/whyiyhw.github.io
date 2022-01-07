---
title: go design (一) array
date: 2022-01-07 14:24:03
categories: golang 
tags:
- array
- 《go语言设计与实现》
---

# golang 中 array 的实现

数组是由相同类型元素的集合组成的数据结构，计算机会为数组分配一块连续的内存来保存其中的元素，我们可以利用数组中元素的索引快速访问特定元素，常见的数组大多都是一维的线性数组。


## `golang` 中 对于数组的实现 `array`

```go
// cmd/compile/internal/types/type.go

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

// Array contains Type fields specific to array types.
type Array struct {
	Elem  *Type // element type
	Bound int64 // number of elements; <0 if unknown yet
}
```

- 由源码可知 golang 中的 array 由 elem（任意类型元素） 跟 bound （边界，长度） 组成
- 所以说 对于 申明长度不一致 array 是不可比较的

## array 的初始化

```go
	arr1 := [3]int{1, 2, 3}
	arr2 := [...]int{1, 2, 3}
	var arr3 = [3]int{1, 2, 3}
```

- 对于 [...] 声明的数组，底层会自动推导 `bound` 并转成 上一种声明方式
- 在**不考虑逃逸分析的情况下**，如果数组中元素的个数小于或者等于 4 个，那么所有的变量会直接在栈上初始化，如果数组元素大于 4 个，变量就会在静态存储区初始化然后拷贝到栈上，这些转换后的代码才会继续进入中间代码生成和机器码生成两个阶段

## 访问与赋值

- 数组的长度固定，那么对于简单的 越界操作 编译时就能发现

  ```go
  arr2 := [...]int{1, 2, 3}
  arr2[3] = 2
  //invalid array index 3 (out of bounds for 3-element array)
  //Compilation finished with exit code 2
  ```

- 而对于 变量 去替代索引的情况，就只能由 运行时去判定

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

  - 很显然 数组中每一位的地址 相差为 10 进制的 8 
  - 那就是说 int  为 8 位

  

  数组是程序中最基础的结构之一，知道索引时，其 `O(1)` 的读写速度也十分优秀，就是需要连续内存这点上，会让使用者比较纠结，其定长的特性，注定了灵活性的不足。`goalng` 中 我们用的最多的 `slice` ，但其底层也是依赖于`array`。
  
  下一篇, 我们一起看看 `slice` 的设计与实现。
