---
title: "Golang Native开发"
date: 2020-12-12T15:56:48+08:00
tags: ["golang",]
categories: ["编程语言"]
---

使用`golang`进行`Android native` 开发  

<!--more-->

## 环境 

-   [x] `android` 开发环境
-   [x] `Android ndk`开发环境
-   [x] `golang` 开发环境
-   [x] `jna` [https://github.com/java-native-access/jna](https://github.com/java-native-access/jna)

## `golang` 代码 

>   这里使用的是`nps`的客户端`sdk`代码 以下是部分函数

```go
//export GetClientStatus
func GetClientStatus() int {
	return client.NowStatus
}

//export CloseClient
func CloseClient() {
	if cl != nil {
		cl.Close()
	}
}
```

## 编译脚本

```shell
#!/usr/bin/env bash

# 设置os为Android
export GOOS=android
export GOARCH=arm
export CGO_ENABLED=1

ndk_tool_path=${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/darwin-x86_64/bin

export CC=${ndk_tool_path}/armv7a-linux-androideabi28-clang
export CXX=${ndk_tool_path}/armv7a-linux-androideabi28-clang++

go build -ldflags "-s -w"  -buildmode=c-shared -o libnpc.so cmd/npc/sdk.go
```

这里生成的是`arm64`**如果需要其他平台，修改`goarch`及对应的`cc`和`cxx`编译器**

## 执行编译 

可以看到生成两个文件 `libnpc.h` 和 `libnpc.so`, 这个库文件有点大 12m

```shell
file libnpc.so
libnpc.so: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, Go BuildID=MDuIWOynaHxgCjOzC_U5/1ePM-5jyxRPWgmWfSKDo/kSP5ZQo9ATT4Whkh9Cyl/sYvwEF0pawhTir_lw0vB, stripped
```

## `Android` 使用

创建`Android project`

1.  更新依赖 

```groovy
    implementation group: 'net.java.dev.jna', name: 'jna-platform', version: '5.6.0'
    implementation group: 'net.java.dev.jna', name: 'jna', version: '5.6.0'
```

2.  复制`libnpc.so` 到`app/libs/arm64-v8a` 目录下 
3.  编辑`build.gradle` 文件，增加

```groovy
    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }
```

4.  解压`jna/dist/android-aarch64.jar` 文件 ，复制`libjnidispatch.so ` 到 `app/libs/arm64-v8a` 目录下 
5.  编写实现, 正常调用执行即可

```java
import com.sun.jna.Library;
import com.sun.jna.Native;

public interface NpcClient extends Library {

    NpcClient ins = Native.load("npc", NpcClient.class);

    /**
     * 对应golang中的方法
     *
     * @param serverAddr 服务端地址
     * @param verifyKey  校验key
     * @param proxyUrl   二层代理的地址
     * @return
     */
    int StartClientByVerifyKey(String serverAddr, String verifyKey, String connType, String proxyUrl);


    /**
     * client 的 status
     *
     * @return
     */
    int GetClientStatus();

    /**
     * 关闭
     */
    void CloseClient();

    /**
     * version
     *
     * @return
     */
    String Version();

    /**
     * logs
     *
     * @return
     */
    String Logs();
}
```

6. 最终的结构 

```
.
├── build.gradle
├── libs
│   └── arm64-v8a
│       ├── libjnidispatch.so
│       ├── libnpc.h
│       └── libnpc.so
├── proguard-rules.pro
└── src
    ├── androidTest
    │   └── java
    │       └── com
    │           └── weixuan
    │               └── npc
    │                   └── ExampleInstrumentedTest.java
    ├── main
    │   ├── AndroidManifest.xml
    │   ├── java
    │   │   └── com
    │   │       └── weixuan
    │   │           └── npc
    │   │               └── SettingsActivity.java
    │   └── res
    │       ├── drawable
    │       │   └── ic_launcher_background.xml
    │       ├── drawable-v24
    │       │   └── ic_launcher_foreground.xml
    │       ├── layout
    │       │   └── settings_activity.xml
    │       ├── mipmap-anydpi-v26
    │       │   ├── ic_launcher.xml
    │       │   └── ic_launcher_round.xml
    │       ├── mipmap-hdpi
    │       │   ├── ic_launcher.png
    │       │   └── ic_launcher_round.png
    │       ├── mipmap-mdpi
    │       │   ├── ic_launcher.png
    │       │   └── ic_launcher_round.png
    │       ├── mipmap-xhdpi
    │       │   ├── ic_launcher.png
    │       │   └── ic_launcher_round.png
    │       ├── mipmap-xxhdpi
    │       │   ├── ic_launcher.png
    │       │   └── ic_launcher_round.png
    │       ├── mipmap-xxxhdpi
    │       │   ├── ic_launcher.png
    │       │   └── ic_launcher_round.png
    │       ├── values
    │       │   ├── arrays.xml
    │       │   ├── colors.xml
    │       │   ├── strings.xml
    │       │   └── styles.xml
    │       └── xml
    │           └── root_preferences.xml
    └── test
        └── java
            └── com
                └── weixuan
                    └── npc
                        └── ExampleUnitTest.java
```



