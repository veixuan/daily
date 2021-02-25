---
# 常用定义
title: "Bind"
date: 2021-02-25T23:19:18+08:00
lastmod: 2021-02-25T23:19:18+08:00
tags: ["cpp", "popular"] 
categories: ["cpp"]             
author: "Neo"          
# 用户自定义
# 你可以选择 关闭(false) 或者 打开(true) 以下选项
comment: false   # 关闭评论
toc: true       # 关闭文章目录
reward: true	 # 关闭打赏
---

`std::bind`的基本使用

<!--more-->

## 使用说明

>   根据已有的`可调用对象`，`构造出新的可调用对象`

核心功能: 动态生成可调用函数

## 静态函数 

```c++
// vector -> str
std::string vec_to_str(const std::string &delim, const std::vector<std::string> &vec) {
	std::string res("[");
	
	for (int i = 0; i < vec.size(); ++i) {
		res.append(vec[i]);
		if (i != vec.size() - 1) {
			res.append(delim);
		}
	}
	res.append("]");
	return res;
}

// std::bind 
// 第一个参数是当前函数 
// 后面是参数，可以直接传参 也可以使用占位符
auto ff = std::bind(vec_to_str, ",", std::placeholders::_1);
std::cout << ff(demo) << std::endl;
```

## 类成员函数 

```c++
	// 绑定类成员函数
	// 第一个参数必须是当前对象指针或者this指针
	entity ent;
	auto set_func = std::bind(&entity::setAge, &ent, std::placeholders::_1);
	set_func(100);
	std::cout << "age = " << ent.getAge() << std::endl;
```

## 可调用对象 

简单来说就是支持`()`方式调用的,有以下几种类型 

-   [ ] 普通函数 
-   [ ] 类成员函数 
-   [ ] 类静态函数 
-   [ ] 仿函数 (类重载了 `operator()` 操作符)
-   [ ] 函数指针
-   [ ] lambda表达式 
-   [ ] std::function 
-   [ ] std::bind 表达式 

