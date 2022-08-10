# C 程序的内存布局

理解C程序的内存布局非常重要。这篇文章将说明C程序的内存布局，并通过一些工具和程序来验证它们。

内存布局是与平台相关的，这篇文章的测试的环境为：

- Ubuntu 20.04 64位
- gcc version 9.3.0

## 预备知识

在开始之前，需要了解`objdump`工具的基本使用，如果你已经非常熟悉它们，请跳过这部分。

我们将用到`objdump -t <objfile>`命令，它将打印`objfile`的符号表，下面是它的输出格式之一：

~~~bash
00000000 l    d  .bss   00000000 .bss
00000000 g       .text  00000000 fred
~~~

- 第1个字段是符号的值
- 第2个字段是字符和空格的集合，它们标识设置在符号上的标志位
- 第3个字段是与符号关联的段（section）
- 第4个字段对于普通的符号是对齐，对于其他符号是大小
- 第5个字段是符号名

在这篇文章中我们只需要关心第3个和第5个字段，我们还需要关心第3个字段的以下值：

- .text表示文本段（text segment）
- .data表示读写数据段（data segment）
- .rodata表示只读数据段（read-only data segment）
- .bss表示未初始化的数据段（block started by symbol）

关于`objdump`工具更详细的解释，请参考objdump(1)。

## 典型的C程序内存布局

一个典型的C程序由5个部分组成，如下图所示：

![memoryLayoutC](https://media.geeksforgeeks.org/wp-content/uploads/memoryLayoutC.jpg)

上面的图片来源于网络。

下面从低地址到高地址依次解释。

### 1. 文本段

文本段（text segment），也被叫做代码段（code segment），它包含了**可执行的指令**。

通常文本段是被共享的，并且它是只读的，从而避免里面的指令被意外修改。

### 2. 初始化的数据段

初始化的数据段（initialized data segment）通常叫做数据段（data segment），它包含了**被初始化的全局变量和静态变量**（准确来说是初始化为非0值）。

数据段其实由两部分组成：只读数据区（read-only area）和读写数据区（read-write area）。

### 3. 未初始化的数据段

未初始化的数据段（uninitialized data segment），通常叫做BSS段。它包含了**被初始化为0或未被显式初始化的全局变量和静态变量**。

### 4. 堆

堆区通常是动态内存分配发生的地方。

堆区通常朝高地址的方向增长。

### 5. 栈

栈区通常是自动变量存放的地方。

栈区通常与堆区连接，并向相反方向增长。当栈指针遇到堆指针时，表明可用的内存已耗尽。

堆区通常朝0地址的方向增长。

## 验证

### 简单的测试程序

下面是一个简单的测试程序，它从低地址到高地址输出内存布局信息：

~~~c
#include <malloc.h>
#include <stdio.h>
#include <string.h>

// .text
void func1() {}
void func2() {}

// .rodata
const int cgi0 = 0x00;
const int cgi1 = 0x01;

// .data
int gi1 = 0x01;

// .bss
int gi0 = 0x00;
int gu;

int main(int argc, char* argv[]) {
    printf("------------ Text segment(.text) ------------\n");

    printf("func1:   %p\n", &func1);
    printf("func2:   %p\n", &func2);

    printf("------------ Initialized data segment(.rodata) ------------\n");

    // .rodata
    static const int cli0 = 0x00;
    static const int cli1 = 0x01;

    printf("cgi1:    %p\n", &cgi1);
    printf("cgi0:    %p\n", &cgi0);
    printf("cli0:    %p\n", &cli0);
    printf("cli1:    %p\n", &cli1);

    printf("------------ Initialized data segment(.data) ------------\n");

    // .data
    static int li1 = 0x01;

    printf("gi1:     %p\n", &gi1);
    printf("li1:     %p\n", &li1);

    printf("------------ Uninitialized data segment(.bss) ------------\n");

    // .bss
    static int li0 = 0x00;
    static int lu;

    printf("gi0:     %p\n", &gi0);
    printf("li0:     %p\n", &li0);
    printf("lu:      %p\n", &lu);
    printf("gu:      %p\n", &gu);

    printf("------------ Heap ------------\n");

    int* var_s1 = (int*)malloc(sizeof(int));
    int* var_s2 = (int*)malloc(sizeof(int));

    // Ensure malloc() is successful
    if (var_s1 && var_s2) {
        printf("*var_h1: %p\n", &(*var_s1));
        printf("*var_h2: %p\n", &(*var_s2));
    } else {
        printf("malloc() error!\n");
    }

    printf("------------ Stack ------------\n");

    printf("var_s1:  %p\n", &var_s1);
    printf("var_s2:  %p\n", &var_s2);

    return 0;
}
~~~

可能的输出：

~~~bash
------------ Text segment(.text) ------------
func1:   0x55f2fbd451a9
func2:   0x55f2fbd451b4
------------ Initialized data segment(.rodata) ------------
cgi1:    0x55f2fbd4600c
cgi0:    0x55f2fbd46008
cli0:    0x55f2fbd4621c
cli1:    0x55f2fbd46220
------------ Initialized data segment(.data) ------------
gi1:     0x55f2fbd48010
li1:     0x55f2fbd48014
------------ Uninitialized data segment(.bss) ------------
gi0:     0x55f2fbd4801c
li0:     0x55f2fbd48020
lu:      0x55f2fbd48024
gu:      0x55f2fbd48028
------------ Heap ------------
*var_h1: 0x55f2fdaab6b0
*var_h2: 0x55f2fdaab6d0
------------ Stack ------------
var_s1:  0x7ffddf88bd58
var_s2:  0x7ffddf88bd60
~~~

### 通过objdump验证

`func1`和`func2`放在文本段：

~~~bash
ocfbnj@ubuntu-server:~$ objdump -t ./memory_layout | grep -E 'func1|func2'
00000000000011a9 g     F .text  000000000000000b              func1
00000000000011b4 g     F .text  000000000000000b              func2
~~~

`cgi1`、`cgi0`、`cli0`和`cli1`放在只读数据段：

~~~bash
ocfbnj@ubuntu-server:~$ objdump -t ./memory_layout | grep -E 'cgi1|cgi0|cli0|cli1'
000000000000221c l     O .rodata        0000000000000004              cli0.2578
0000000000002220 l     O .rodata        0000000000000004              cli1.2579
0000000000002008 g     O .rodata        0000000000000004              cgi0
000000000000200c g     O .rodata        0000000000000004              cgi1
~~~

`gi1`和`li1`放在读写数据段：

~~~bash
ocfbnj@ubuntu-server:~$ objdump -t ./memory_layout | grep -E '\<gi1|\<li1'
0000000000004014 l     O .data  0000000000000004              li1.2580
0000000000004010 g     O .data  0000000000000004              gi1
~~~

`gi0`、`gu`、`li0`和`lu`放在BSS段：

~~~bash
ocfbnj@ubuntu-server:~$ objdump -t ./memory_layout | grep -E '\<gi0|gu|\<li0|lu'
0000000000004020 l     O .bss   0000000000000004              li0.2581
0000000000004024 l     O .bss   0000000000000004              lu.2582
0000000000004028 g     O .bss   0000000000000004              gu
000000000000401c g     O .bss   0000000000000004              gi0
~~~

## 参考

- objdump(1)
- [Memory Layout of C Programs - GeeksforGeeks](https://www.geeksforgeeks.org/memory-layout-of-c-program/)
