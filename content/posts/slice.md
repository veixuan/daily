---
# 常用定义
title: "slice"
date: 2020-12-30T23:19:39+08:00
lastmod: 2020-12-30T23:19:39+08:00
tags: ["golang"] 
categories: ["golang"]             
author: "Neo"          
# 用户自定义
# 你可以选择 关闭(false) 或者 打开(true) 以下选项
comment: false   # 关闭评论
toc: true       # 关闭文章目录
reward: true	 # 关闭打赏
---

`slice`的使用和原理

<!--more-->



## 结构 

```go
// /usr/local/go/src/runtime/slice.go:13
// go version go1.16beta1
type slice struct {
	array unsafe.Pointer // 指向一块连续的内存 存储的是实际的数据 
	len   int            // slice的长度 
	cap   int            // slice的容量 
}
```

## 使用

### 创建 

```go
	ints := make([]int, 0, 2)

	var strs []string
```

### 追加

```go
ints := make([]int, 0, 2)
ints = append(ints, 1)
ints = append(ints, 2)
ints = append(ints, 3)
ints = append(ints, 4, 5, 6)
```

### 遍历

```go
	// for range 
	for index, value := range ints {
		
	}
```

### 扩容 

```go
// runtime.growslice
// et slice中元素的类型
// old 旧的slice 
// cap 扩容需要的最小容量
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.cap < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

  // 如果旧的cap < 1024 则新的cap = old.cap * 2 
  // 如果旧的cap >= 1024 则新的cap =  old.cap * 1.25 
  
  // 计算内存对齐
	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
    // et中的元素大小是1，
	case et.size == sys.PtrSize:
    // lenmem = 6 * 8 = 48 
		lenmem = uintptr(old.len) * sys.PtrSize
    // bewlenmem = 6 * 2 * 8 = 96 
		newlenmem = uintptr(cap) * sys.PtrSize
    // capmem = roundupsize(96) = 96 （两次从数组中计算出来 ）
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
    // false 
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
    // newcap = 96 / 8 = 12
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
		}
	}
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}

// 测试代码 

	ints := make([]int, 0, 6)
	ints = append(ints, 1, 2, 3, 4, 5, 6)
	log.Printf("ints len %d cap %d", len(ints), cap(ints))
// 触发扩容 
	ints = append(ints, 7)
	log.Printf("ints len %d cap %d", len(ints), cap(ints))
```

> 说明:
>
> 1.  参数分别是slice的类型、slice、扩容需要的最小容量，当前cap=2、len=2，再append 一个元素，那么扩容需要的最小容量是5
> 2.  如果旧的cap < 1024 则新的cap = old.cap * 2 
> 3.  如果旧的cap >= 1024 则新的cap =  old.cap * 1.25 
> 4.  最终新的cap大小还需要做一次内存对齐之后确定 
> 5.  参考上述的测试代码 计算过程 

**如果扩容之后，还没有触及原数组的容量，那么，切片中的指针指向的位置，就还是原数组，如果扩容之后，超过了原数组的容量，那么，Go就会开辟一块新的内存，把原来的值拷贝过来，这种情况丝毫不会影响到原数组**

### 克隆 

```go
	newInts := make([]int, len(ints))
	copy(newInts, ints)
```

### 元素更新 

```go
// 删除 第三个元素 
ints = append(ints[0:2], ints[3:]...)
```

## 比较 

不能比较  `golang`中包含 `slice`、`func`、`map` 等类型 不支持`comare`

## `nil slice` vs `empty slice`

nil slice中的底层指针也是nil，empty slice 中的底层指针不为nil。使用时没有区别，都可以在append时初始化。

## 是否线程安全 

非线程安全，但是不会像map一样直接panic。在多线程环境中，使用加锁保证线程安全 

## 性能优化

**提前预分配长度**





 