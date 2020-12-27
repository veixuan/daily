---
# 常用定义
title: "channle的使用场景"
date: 2020-12-28T00:01:37+08:00
lastmod: 2020-12-28T00:01:37+08:00
tags: ["golang"] 
categories: ["golang"]             
author: "Neo"          
# 用户自定义
# 你可以选择 关闭(false) 或者 打开(true) 以下选项
comment: false   # 关闭评论
toc: true       # 关闭文章目录
reward: true	 # 关闭打赏
---

`channel`的使用案例

<!--more-->

## 信号传递 

```go
	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	select {
	case sg := <-sigs:
		log.Printf("shutdown with signal %v", sg)
		log.Printf("will exit in five seconds ...")
		time.Sleep(time.Second * 5)
	}

// 可以控制进程在退出之前做一些收尾工作 
```

## 超时控制

>   和timer搭配一起用   
>
>   当然有更优雅的实现方式`contextwithtimeout` 这里只是举例 也可以实现类似功能 

```go
func Get(url string, timeout time.Duration) (status int) {
	done := make(chan struct{})
	go func() {
		defer close(done)
		resp, err := http.Get(url)
		if err != nil {
			status = -1
		}
		if resp != nil {
			status = resp.StatusCode
			resp.Body.Close()
		}
	}()

	select {
	case <-done:
		return
	case <-time.After(timeout):
		return -100
	}
}
```

## 定时任务 

```go
func Loop() {
	ticker := time.Tick(1 * time.Second)
	go func() {
		for {
			select {
			case <-ticker:
				log.Printf("now is %v", time.Now().String())
			}
		}
	}()
}
```

## 消息传递 

>   可以实现类似java 中future的功能 

## 控制并发 

用chan控制go routine的数量，当队列处理不过来时，直接丢弃 





