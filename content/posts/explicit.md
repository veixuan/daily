---
# 常用定义
title: "explicit"
date: 2021-03-06T14:15:29+08:00
lastmod: 2021-03-06T14:15:29+08:00
tags: ["cpp", "popular"] 
categories: ["cpp"]             
author: "Neo"          
# 用户自定义
# 你可以选择 关闭(false) 或者 打开(true) 以下选项
comment: false   # 关闭评论
toc: true       # 关闭文章目录
reward: true	 # 关闭打赏
---

`explicit`关键字的使用 

<!--more-->

## 作用 

1.  只能用来修饰构造函数 
2.  禁止隐式调用拷贝构造函数
3.  禁止对象间的隐式转换

## 隐式转换

```c++
class Score {

public:
	explicit Score(double score) : _score(score) {
		std::cout << "init with " << score << std::endl;
	}
	
	
	static void info(Score score) {
		std::cout << "info-> " << score._score << std::endl;
	}

private:
	double _score;
};
```

然后这样调用

```c++
Score::info(2.0);
// 会先将2.0 调用构造函数 转换成 score 对象，然后再执行对应的方法 
```

## explicit

增加`explicit`修饰之后，再次这样调用，就会编译失败

```c++
	explicit Score(double score) : _score(score) {
		std::cout << "init with " << score << std::endl;
	}
// Score::info(2.0);
// error: no viable conversion from 'double' to 'Score'
```

## 优点 

可以避免 编译器执行 非预期的类型转换