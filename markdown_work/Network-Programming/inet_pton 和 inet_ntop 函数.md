# inet_pton 和 inet_ntop 函数

inet_pton和inet_ntop是两个新函数，它用来代替inet_aton、inet_addr和inet_ntoa函数。inet_pton和inet_ntop适用于IPv4和IPv6地址。

## inet_pton

~~~c
#include <arpa/inet.h>

int inet_pton(int af, const char *src, void *dst);
~~~

这个函数转换字符串`src`到一个使用`af`地址族的网络地址结构，然后拷贝该网络地址结构到`dst`。参数`af`必须为AF_INET或AF_INET6。`dst`以网络字节序写入。

下面是当前支持的地址族：

- AF_INET
  - `src`指向一个点分十进制格式的IPv4网络地址的字符串，“ddd.ddd.ddd.ddd”，`ddd`是一个十进制数，范围为0-255。该地址被转换为一个`struct in_addr`然后被拷贝到`dst`，`dst`必须具有`sizeof(struct in_addr)`（4）个字节的长度。
- AF_INET6
  - `src`指向一个包含IPv6网络地址的字符串，该地址被转换为一个`struct in6_addr`然后被拷贝到`dst`，`dst`必须具有`sizeof(struct in6_addr)`（16）个字节的长度。

`inet_pton()`在成功时返回1。如果`src`没有包含一个有效的网络地址则返回0。如果`af`不是一个有效的地址族，返回-1并将`errno`设置为EAFNOSUPPORT。

## inet_ntop

~~~c
#include <arpa/inet.h>

const char *inet_ntop(int af, const void *src,
                      char *dst, socklen_t size);
~~~

这个函数转换使用`af`地址族的网络地址结构`stc`到一个字符串。结果字符串被拷贝到`dst`指向的缓冲区中，它必须不为空指针。调用者通过`size`参数指定缓冲区中可用的字节数。

下面是当前支持的地址族：

- AF_INET
  - `src`指向一个`struct in_addr`（网络字节序），它被转换成点分十进制格式的IPv4网络地址“ddd.ddd.ddd.ddd”。缓冲区`dst`至少为INET_ADDRSTRLEN字节的长度。
- AF_INET6
  - `src`指向一个`struct in6_addr`（网络字节序），它被转换成最合适的IPv6网络地址格式。缓冲区`dst`至少为INET6_ADDRSTRLEN字节的长度。

`inet_ntop()`在成功时返回一个非空的指向`dst`的指针。如果发生错误则返回NULL。当`af`不是一个有效的地址族时，`errno`被设置为EAFNOSUPPORT，当转换后的地址字符串将超过`size`给定的大小时，`errno`被设置为ENOSPC。

## 例子

~~~c
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char* argv[]) {
    unsigned char buf[sizeof(struct in6_addr)];
    int domain, s;
    char str[INET6_ADDRSTRLEN];

    if (argc != 3) {
        fprintf(stderr, "Usage: %s {i4|i6|<num>} string\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    domain = (strcmp(argv[1], "i4") == 0) ? AF_INET :
             (strcmp(argv[1], "i6") == 0) ? AF_INET6 : atoi(argv[1]);

    s = inet_pton(domain, argv[2], buf);
    if (s <= 0) {
        if (s == 0) {
            fprintf(stderr, "Not in presentation format");
        } else {
            perror("inet_pton");
        }
        exit(EXIT_FAILURE);
    }

    if (inet_ntop(domain, buf, str, INET6_ADDRSTRLEN) == NULL) {
        perror("inet_ntop");
        exit(EXIT_FAILURE);
    }

    printf("%s\n", str);

    exit(EXIT_SUCCESS);
}
~~~

输入：

~~~bash
$ ./a.out i6 0:0:0:0:0:0:0:0
$ ./a.out i6 1:0:0:0:0:0:0:8
$ ./a.out i6 0:0:0:0:0:FFFF:204.152.189.116
~~~

输出：

~~~bash
::
1::8
::ffff:204.152.189.116
~~~

## 参考

- inet_pton(3)
- inet_ntop(3)
