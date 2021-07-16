---
layout: post
title: GDB list无法查看代码
category : C
tags : [FreeSwitch]
---
GDB list无法查看代码

gdb调试代码，通过list命令可以查看当前文件的代码信息，有时候会遇到不能显示代码的情况：
```commandline
(gdb) l
1	test.c: No such file or directory.
	in test.c
(gdb)
1	in test.c
```

如上，显示 No such file ... 是因为编译时的源码路径发生变化了。这是因为gdb -g参数在编译时，并不会把源码附加到编译结果中，而是在编译结果中记录了源码的位置。
如果编译完成后，删除了源码或者移动了源码位置，那么就会导致不能显示函数信息。

举例：
```c
test.c
#include <stdio.h>
#include <stdlib.h>

int main()
{
	printf("hello world\n");
	return 0;
}

gcc -g -o test test.c
```
以上，编译完成后，在当前目录下创建一个新的目录，./src, 执行 mv test.c ./src 之后执行 gdb ./test ，通过 list 就会发现无法显示源码
解决方法：mv src/test.c . 即可