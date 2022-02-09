# C 语言常用库函数-sprintf


## 函数原型 {#函数原型}

```c
#include <stdio.h>

/*
函数说明:   按照format指定的格式填充字符串str，以 '\0' 结尾。
函数返回值: 若成功则返回存入数组的字符数，若编码出错则返回负值。
*/
int sprintf(char *str, const char *format, ...);

```


## 用法分析 {#用法分析}

sprintf 一般用来把其他类型的数据转换成字符数组储存。它会自动在所写入字符组的末尾加上 NUL 字符来表示字符串的结束，NUL 字符不被计入返回值。

sprintf 和 strcpy 一样，属于无边界检查的一类函数，不建议使用。


## 注意事项 {#注意事项}

对于新写的代码，请抛弃 sprintf，用 snprintf 代替。

