---
title: "go中struct能否比较么?"
date: 2020-10-31T00:38:32+08:00
tags: ["golang",]
categories: ["编程语言"]
toc: true 
---

`Struct`在`golang`中能否比较?
<!--more-->

## 哪些类型不能比较

1.  slice
2.  map
3.  function
4.  以及包含了上述三种类型之一的struct

## 为什么这些类型不能比较

尝试比较

```bash
invalid operation: a1 == a2 (slice can only be compared to nil)
invalid operation: a1 == a2 (map can only be compared to nil)
invalid operation: a1 == a2 (func can only be compared to nil)
```

从程序的实现角度来讲，类似于底层没有实现 `==` 操作符

另一方面，这三种类型的相等 有歧义，所以未实现

## interface、channel 可比较

1.  channel。只有在它们引用着相同的底层内部通道或者它们都为nil时比较结果才为true
2.  interface。只有这两个接口值的动态类型相同、动态类型为可比较类型、并且动态值相等 或者都为nil 才为true

## 如果要比较slice、map时如何比较？

```go
reflect.DeepEqual
func DeepEqual(x, y interface{}) bool {
	if x == nil || y == nil {
		return x == y
	}
	v1 := ValueOf(x)
	v2 := ValueOf(y)
	if v1.Type() != v2.Type() {
		return false
	}
	return deepValueEqual(v1, v2, make(map[visit]bool), 0)
}
```

结论：如果struct中包含了 上述不能比较的类型，是不能直接比较的。

## 终极大招 

>   如何保证一个自定义类型`struct` 不能被比较?

```go
// 1. 定义一个no compare 的struct 
type noCompare struct {
	nocmp [0]func() // 同时包含了slice和function
}
// 2. 将这个struct 组合到自定义类型中 
type Dummy struct {
	noCompare
}
// 3. 尝试比较将直接报错 
	d1 := Dummy{}
	d2 := Dummy{}
	log.Printf("cmp %+v", d1 == d2)
// 输出
// invalid operation: d1 == d2 (struct containing noCompare cannot be compared)
```



