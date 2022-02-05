# C 语言常用库函数-snprintf


## 函数原型 {#函数原型}

```c
#include <stdio.h>
/*
函数说明:   最多从源串中拷贝size－1个字符到目标串中，然后再在后面加一个0。所
           以如果目标串的大小为size的话，将不会溢出。
函数返回值: 若成功则返回存入数组的字符数，若编码出错则返回负值。
*/
int snprintf(char *str, size_t size, const char *format, ...);

```


## 用法分析 {#用法分析}

snprintf 和 sprintf 不同的是，当缓冲区不够用时，snprintf 会返回一个大于等于 n 的值，出
错时返回一个负值。因此，当返回值是一个不大于 n-1 的非负值时，它可以保证缓冲区是以
NUL 结尾。

snprintf 的正确用法:

```c
#include <stdio.h>

#define BUFSIZ 16

int main(int argc,char **argv)
{
    char buf[BUFSIZ];

    /* 注意这里长度不需要用sizeof(buf)-1，因为snprintf只会拷贝size-1个字节，
     * 并自己加上NULL结尾
     */
    snprintf(buf, sizeof(buf), "%s", argv[1]);

    return 0;
}
```


## 注意事项 {#注意事项}

不管是 sprintf 还是 snprintf，在使用的时候必须注意被拷贝的字串于作为参数的字串不能
相同，因为这会导致不可预知的返回值。例如：

```c
#include <stdio.h>

int main(int argc, char **argv)
{
    char str[16] = "hello";

    snprintf(str, sizeof(str), "%s-%d", str, 1);

    printf("%s\n", str);

    return 0;
}
```

参考 [这份资料](https://pubs.opengroup.org/onlinepubs/000095399/functions/printf.html)

```sh
If copying takes place between objects that overlap as a result of a call to
sprintf() or snprintf(), the results are undefined.
```

