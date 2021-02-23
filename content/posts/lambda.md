---
# 常用定义
title: "Lambda表达式"
date: 2021-02-23T23:19:30+08:00
lastmod: 2021-02-23T23:19:30+08:00
tags: ["cpp", "popular"] 
categories: ["cpp"]             
author: "Neo"          
# 用户自定义
# 你可以选择 关闭(false) 或者 打开(true) 以下选项
comment: false   # 关闭评论
toc: true       # 关闭文章目录
reward: true	 # 关闭打赏
---

c++ 11的新特性，支持函数式编程。

<!--more-->

## 说明 

>   ```
>   匿名函数。可以像对象一样使用，也可以和函数一样进行求值运算
>   ```

## 定义格式

```c++
[capture list] (params list) mutable exception-> return type { 
  function body 
}
```

>   说明

-   `capture list` 捕获的外部变量列表 
-   `params list` 形参列表 
-   `mutable` 指示符，用来标识是否可以修改外部的变量 
-   `exception` 异常设定 
-   `return type` 返回值类型 
-   `function body ` 函数体

其中`mutable`、`exception`、`return type` 可以按照实际的情况写，不是必须的。即便省略了return type，编译器可以根据函数体重是否有return语句来推断会lambda表达式的返回值。

## 实际的案例 

```c++
// 排序 
void sort_vec(std::vector<int> &input) {
	std::sort(input.begin(), input.end(), [](int a, int b) -> bool {
		return a < b;
	});
}
```

## 变量捕获

| 格式          | 说明                                        |
| ------------- | ------------------------------------------- |
| `[]`          | 不捕获任何外部变量                          |
| `[&]`         | 以引用的方式 捕获所有的外部变量             |
| `[=]`         | 以值的方式，铺货所有的外部变量              |
| `[this]`      | 以值的方式，捕获this指针                    |
| `[p1,p2,...]` | 以值的方式，捕获p1/p2等后续的其它参数       |
| `[&p1,&p2]`   | 以引用的方式，捕获p1/p2变量                 |
| `[=, &x]`     | 变量x以引用形式捕获，其余变量以传值形式捕获 |
| `[&, x]`      | 变量x以值的形式捕获，其余变量以引用形式捕获 |
| `[&p1,p2]`    | P1以引用的方式捕获，p2以值的方式捕获        |

## 修改捕获变量

如果以值传递参数，默认是不能修改传递的参数的。但是可以加上`mutable`关键字，来修改捕获的外部值参数

```c++
	int sum = 0;
	
	// sum是引用传递 可以修改
	auto sumer = [&sum](int x) {
		sum += x;
	};
	
	std::for_each(src.begin(), src.end(), sumer);
	std::cout << "sum = " << sum << std::endl;
	

	// 引用所有外部变量
	auto to_str = [&]() -> std::string {
		return std::to_string(sum);
	};
	
	std::cout << to_str();


```

## 参数说明

1.  参数列表不支持默认参数
2.  不支持可变参数 
3.  所有参数要有参数名 

## lambda 表达式作为函数参数

```c++
// std::function<std::string ()>
// 代表一个 返回值是 string，无参数列表的函数
void exec_lambda_func(std::function<std::string()> const &f) {
	std::cout << "exec " << f() << std::endl;
}

	// 引用所有外部变量
	auto to_str = [&]() -> std::string {
		return std::to_string(sum);
	};
	
	exec_lambda_func(to_str);
```

