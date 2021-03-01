---
# 常用定义
title: "Constructor"
date: 2021-03-01T22:48:57+08:00
lastmod: 2021-03-01T22:48:57+08:00
tags: ["cpp", "popular"] 
categories: ["cpp"]             
author: "Neo"          
# 用户自定义
# 你可以选择 关闭(false) 或者 打开(true) 以下选项
comment: false   # 关闭评论
toc: true       # 关闭文章目录
reward: true	 # 关闭打赏
---

`c++ 2.0` 中的类六大函数 

<!--more-->

## 类基本函数

```c++
//
// Created by 韦轩 on 2021/3/1.
//


#pragma once

#include <iostream>
#include <string>

class Person {
public:
	/**
	 * 编译器生成的构造函数
	 */
	Person() = default;
	
	/**
	 * 自定义的构造函数
	 * @param size
	 * @param data
	 */
	Person(int size, int *data);
	
	/**
	 * 拷贝构造
	 * @param other
	 */
	Person(const Person &other);
	
	/**
	 * 拷贝赋值(其实是赋值运算符重载)
	 * @param other
	 * @return
	 */
	Person &operator=(const Person &other);
	
	/**
	 * 移动构造
	 * @param other
	 */
	Person(Person &&other);
	
	/**
	 * 移动赋值
	 * @param other
	 * @return
	 */
	Person &operator=(Person &&other);
	
	/**
	 * 析构函数
	 */
	virtual ~Person();
	
	/**
	 * 辅助打印的函数
	 * @param os
	 * @param person
	 * @return
	 */
	friend std::ostream &operator<<(std::ostream &os, const Person &person);

private:
	int size_;
	int *data_;
};
```

## 构造函数 

>    构造函数，用于初始化参数或者构造一个实例。

编译器会自动生成一个默认的构造函数. 也可以手动指定，使用 `=default`关键字表示需要编译器自动生成的这个。

```c++
// 实现上面自定义的部分 	
Person::Person(int size, int *data) : size_(size), data_(data) {
	std::cout << "-----call constructor---------" << std::endl;
}

	int *data = new int[2];
	data[0] = 100;
	data[1] = 200;
	
	Person person{2, data};
	
	std::cout << person << std::endl;

// -----call constructor---------
// size_: 2 data_: 0->100	1->200	
```

## 拷贝构造函数

>   使用另一个对象来初始化当前对象

```c++
Person::Person(const Person &other) : size_(other.size_), data_(new int[other.size_]) {
	for (int i = 0; i < size_; ++i) {
		this->data_[i] = other.data_[i];
	}
	std::cout << "copy with " << size_ << "\t" << data_ << std::endl;
}

	Person& p2(person);
	std::cout << "p2->" << p2 << std::endl;

// p2->size_: 2 data_: 0->100	1->200	
```

## 拷贝赋值函数 

>   本质是 = 运算符的重载，将另一个对象直接赋值给当前对象 

```c++
Person &Person::operator=(const Person &other) {
	if (this != &other) {
		// 先释放当前对象分配的指针
		delete[] this->data_;
		this->size_ = other.size_;
		this->data_ = new int[other.size_];
		std::copy(other.data_, other.data_ + this->size_, this->data_);
	}
	
	std::cout << "---------------operator= -----------" << std::endl;
	return *this;
}
```

## 移动构造函数

>   主要目的: 为了解决拷贝构造的性能和资源安全性问题。
>
>   在使用原对象对当前对象赋值之后，将原对象的指针置为空。

```c++
Person::Person(Person &&other) : data_(nullptr), size_(0) {
	this->data_ = other.data_;
	this->size_ = other.size_;
	
	// 释放原来的
	other.data_ = nullptr;
	other.size_ = 0;
}

// 测试
	Person p4(std::move(p2));
	std::cout << "p4->" << p4 << std::endl;
	std::cout << "p2->" << p2 << std::endl;

// p4->size_: 2 data_: 0->100	1->200	
// p2->size_: 0 data_: 
```

## 移动赋值函数

>   和上面的功能类似 

```c++
Person &Person::operator=(Person &&other) {
	if (this == &other) {
		return *this;
	}
	delete[] this->data_;
	// 复制
	this->data_ = other.data_;
	this->size_ = other.size_;
	
	// 释放other
	other.data_ = nullptr;
	other.size_ = 0;
}
```

## 析构函数

>   用于释放分配的内存 

```c++
Person::~Person() {
	delete[] this->data_;
}
```

