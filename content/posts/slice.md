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


// 
```



### 扩容 



### 克隆 



### 元素更新 



## `reslice`



## 比较 



## `nil slice` vs `empty slice`



## 是否线程安全 



## 性能优化及陷阱

 