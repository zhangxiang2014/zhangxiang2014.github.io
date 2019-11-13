---
layout: post
title:  mac下nativecrash本地分析流程
date:   2019-11-13 15:13
categories: android
permalink: /archivers/mac_nativecrash_local
---

建议先阅读[Android 平台 Native 代码的崩溃捕获机制及实现
](https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w)
###1.首先本地获得程序crash的.dump文件
nativecrash的dump文件获取可以参考
[极客时间性能优化教学的sample](https://github.com/AndroidAdvanceWithGeektime/Chapter01)

###2.本地编译google breakpad工具用来分析.dump文件
> git clone https://github.com/google/breakpad.git  
./configure  
make

大概率编译出错,分析出错的日志文件config.log发现
> xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun  

解决方式：  

- 输入命令并执行：  
xcode-select --install  
- 这个时候会弹出一个窗口，提示让下载xcode，选择安装重复make

###3.使用minidump_stackwalker 工具来根据 minidump 文件生成堆栈跟踪log
> 
> cd src/processor/  
 ./minidump_stackwalk ../../../Chapter01/crashDump/06842276-c84f-4fb3-a24ba3b3-e43a6b7c.dmp >crash.txt   
    
将dump文件转为txt文件

###4.观察txt文件跟踪log
摘录重点部分如下：
>
>Operating system: Android
                  0.0.0 Linux 4.9.112-perf-g8dcf91a #1 SMP PREEMPT Fri Sep 27 22:09:09 CST 2019 aarch64  
CPU: arm64  
     8 CPUs

>GPU: UNKNOWN  

>Crash reason:  SIGSEGV /SEGV_MAPERR  
>Crash address: 0x0  
>Process uptime: not available  

>Thread 0 (crashed)  
> 0  libcrash-lib.so + 0x650

cpu是arm64，错误类型 SIGSEGV /SEGV_MAPERR，so文件的偏移地址0x650（这个信息最为重要直接对应到实际代码）

###5.符号解析，利用ndk下分析工具分析对应so文件和偏移地址
可以使用 ndk 中提供的addr2line来根据地址进行一个符号反解的过程,该工具在$NDK_HOME/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/arm-linux-androideabi-addr2line  
注意：此处要注意一下平台，如果是 arm64位的 so，解析是需要使用 aarch64-linux-android-4.9下的工具链
>
>build/intermediates/transforms/mergeJniLibs/debug/0/lib/arm64-v8a/libcrash-lib.so 0x650

>结果：  
>Crash()
>/Users/mac/AndroidStudioProjects/Chapter01/sample/.externalNativeBuild/cmake/debug/arm64-v8a/../../../../src/main/cpp/crash.cpp:10

跟踪到对应函数和对应源代码行。  

```c++
#include <stdio.h>
#include <jni.h>


/**
 * 引起 crash
 */
void Crash() {
    volatile int *a = (int *) (NULL);
    *a = 1;
}

extern "C"
JNIEXPORT void JNICALL
Java_com_dodola_breakpad_MainActivity_crash(JNIEnv *env, jobject obj) {
    Crash();
}
```

**引申问题，如何在线上统计nativecrash分析结果？**


