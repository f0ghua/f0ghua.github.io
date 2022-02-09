# C 语言常用库函数-strcpy


## 函数原型 {#函数原型}

```c
#include <string.h>

char *strcpy(char *dest, const char *src);
```


## 用法分析 {#用法分析}

strcpy 属于无边界检查的一类函数，因此使用起来十分的危险，在标准文档中已经不建议使用。

无边界检查很容易就会发生缓冲溢出的错误，例如:

```c
#define BUFSIZ 265
int main(int argc, char **argv)
{
    char buf[BUFSIZ];

    strcpy(buf, argv[1]);

    return 0;
}
```

注意这个例子中 buf 定义了大小为 256，但是 argv[1]却是不限制大小的。由于 strcpy 只知道拷贝到源字串结束，因此一旦 argv[1]的长度大于 BUFSIZ，就会引起缓冲溢出了。


## 注意事项 {#注意事项}

请抛弃 strcpy，用 strncpy 代替，最好用 strlcpy。

