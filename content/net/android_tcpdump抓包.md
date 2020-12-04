---
title: "Android_tcpdump抓包"
date: 2020-12-05T00:41:48+08:00
tags: ["net",]
categories: ["net"]
---

# 背景

`Android`手机`tcpdump`抓包
<!--more-->

>   很多app不是使用`http`、`https`等应用层协议，使用`charles`无法抓到包。

## 环境

-   [x] 小米mix 3，已root
-   [x] Android tcpdump https://www.androidtcpdump.com/android-tcpdump/downloads

## 安装 

```shell
adb push tcpdump /sdcard/data/local/tmp
chmod 777 /sdcard/data/local/tmp/tcpdump
```

## 执行抓包 

```shell
tcpdump -i any -p -s 0 -w capture.pcap

 # "-i any": listen on any network interface

 # "-p": disable promiscuous mode (doesn't work anyway)

 # "-s 0": capture the entire packet

 # "-w": write packets to a file (rather than printing to stdout)
```

## 下载到本地 

```shell
adb pull /storage/emulated/0/data/local/tmp/capture.pcap
```

## 使用`wireshark` 打开 

![](https://i.loli.net/2020/12/05/iyOwxFvGZqkA1p4.png)

