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

附：

以上也可以通过objdump看到段信息来确认,看如下第4~5 行后边显示的 /home/test/data/meng/fs/test 就是我的代码所在的路径
```commandline
$ gcc -c test.c -o  test.o -g
$ objdump -s test.o
1. Contents of section .debug_str:
2. 0000 6c6f6e67 206c6f6e 6720696e 7400756e  long long int.un
3. 0010 7369676e 65642069 6e74006d 61696e00  signed int.main.
4. 0020 2f686f6d 652f776f 726b2f6f 7062696e  /home/test/data
5. 0030 2f6d656e 67616c6f 6e672f66 732f7465  /meng/fs/test
6. 0040 7374006c 6f6e6720 756e7369 676e6564  st.long unsigned
7. 0050 20696e74 00474e55 20432034 2e342e36   int.GNU C 4.4.6
8. 0060 20323031 32303330 35202852 65642048   20120305 (Red H
9. 0070 61742034 2e342e36 2d342900 6c6f6e67  at 4.4.6-4).long
```