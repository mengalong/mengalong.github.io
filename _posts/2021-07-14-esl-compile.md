---
layout: post
title: FreeSwitch ESL 编译
category : C
tags : [FreeSwitch]
---
FreeSwitch ESL 库单独编译

在Mac上安装了最新版的FreeSwitch(1.10.6),尝试使用ESL提供的C接口控制FreeSwitch。ESL的编译有两种方式,这里以官方的 testclient.c 为例：
+ 方法一：在FreeSwitch源码目录下直接使用默认的makefile编译
```c
$ cd /usr/local/src/freeswitch/libs/esl
$ make
$ ./testclient
UP 0 years, 0 days, 8 hours, 54 minutes, 18 seconds, 968 milliseconds, 695 microseconds
FreeSWITCH (Version 1.10.6 -release 64bit) is ready
22 session(s) since startup
0 session(s) - peak 2, last 5min 0
0 session(s) per Sec out of max 30, peak 1, last 5min 0
1000 session(s) max
min idle cpu 0.00/100.00
Current Stack Size/Max 240K/8192K
```
以上可以看到，testclient会默认连接本地localhost的8021端口，查询相关信息并返回

+ 方法二：将testclient.c和原生代码分离，单独编译
    + 创建一个mytest的目录，将testclient.c拷贝进去
    + 编写MakeFile，如下：
```c
ESLPATH = /usr/local/src/freeswitch/libs/esl
CFLAGS = -I$(ESLPATH)/src/include

all: testclient.c
	gcc $(CFLAGS) -o testclient testclient.c /usr/local/src/freeswitch/libs/esl/.libs/libesl.a
```
- 
    + 执行make编译即可
    
遇到的问题：

1. esl编译依赖esl的静态链接库，因此在编译命令中需要指定/usr/local/src/freeswitch/libs/esl/.libs/libesl.a
这个文件是因为本机没有安装libesl，在方法一执行make的时候会再源码目录中生成，需要注意
2. 如果执行 -lesl 参数指定的话，可能会提示如下信息，是因为esl没有安装，用上边的临时方法即可解决
```buildoutcfg
ld: library not found for -lesl
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```
