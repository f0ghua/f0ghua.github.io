# C语言常用库函数-sscanf


## 函数原型 {#函数原型}

```c

#include <stdio.h>

int sscanf(const char *str, const char *format, ...);

```

这里 format 可以是一个或者多个

`{ %[*][width][{h|l|I64|L}]type | ' ' | '\t' | '\n' | 非%符号 }`

其中

| 符号  | 说明             |
|-----|----------------|
| \*    | 添加则表示满足条件的被过滤 |
| width | 表示宽度         |
| h     | 表示单字节       |
| l     | 表示双字节       |
| L     | 表示 4 个字节    |
| I64   | 表示 8 个字节    |
| type  | 如 s, d 之类，表示字符串等 |


## 用法分析 {#用法分析}


### 基本用法 {#基本用法}

sscanf 一般用来从一个 ascii 字符串中读取一些值，基本用法很简单。例如

```c
#include <stdio.h>

int main ()
{
    char *str = "121000-13:00:20";
    int s[6];
    int i;

    sscanf(str,"%2d%2d%2d-%d:%d:%d",
           &s[0],&s[1],&s[2],&s[3],&s[4],&s[5]);

    for(i = 0;i < 6;i++)
        printf("s[%d] = %d\n",i,s[i]);

    return 0;
}
```


### 高级用法 {#高级用法}

sscanf 提供了简单的模式匹配，其格式为 `%[]` 。

这里的 `[ ]` 和正则表达式中是一样的，表示匹配其中出现的字符序列。如果在 `[ ]` 中使用 `^` ，则是表示取反。

例如:

```sh
[a-z]           表示小写字母序列,如abc ...
[^a-z]          表示除小写字母的字符
```

举例来说明

```c
#include <stdio.h>

int main ()
{
    char *str = "12:10:00-13:00:20";
    char s[6][3];
    int i;

    sscanf(str,"%[^:]:%[^:]:%[^-]-%[^:]:%[^:]:%[^:]",
           s[0],s[1],s[2],s[3],s[4],s[5]);

    for(i = 0;i < 6;i++)
        printf("s[%d] = %s\n",i,s[i]);

    return 0;
}
```

这个例子分隔出了 str 字符串中的所有数字。其格式字符串分析如下： `%[^:]` 表示匹配不含 `:` 的字符串,所以第一个 `%[^:]` 就匹配了数字字符串 `12` ,第二个则匹配了
`10` ; 同理， `%[^-]` 表示匹配不含 `-` 的字符串,所以匹配字符串 `13` ；其他的就不言而寓了。

还是上面的例子，如果我们把格式化字串变为
`%[0-9]:%[0-9]:%[0-9]-%[0-9]:%[0-9]:%[0-9]` , 将会如何？嘿嘿，一样的结果。因为
`%[0-9]` 就匹配了数字组成的字符串。

相信这一个例子应该足够说明问题，具体情况具体分析，主要还是靠大家活学活用了。

再举一个复杂点的例子，供参考：

```c
#include   <stdio.h>

/* 获取/和@之间的字符串 */
int main()
{
    const char *s = "iios/12DDWDFF@122";
    char buf[20];

    sscanf(s, "%*[^/]/%[^@]", buf);
    printf("%s\n", buf);

    return 0;
}
```


## 注意事项 {#注意事项}


### 返回值检查 {#返回值检查}

我们在使用 sscanf 的时候往往会忽略其返回值，事实上，sscanf 的返回值是很有用处的。

当匹配出错的时候其返回值为 EOF，否则返回匹配成功的参数个数，如够一个都没匹配到，就是 0 了。

举例子如下：

```c
int main(int argc, char **argv)
{
    int i = 0, j = 0, r;

    r = sscanf(argv[1], "%d:%d", &i, &j);

    printf("r = %d, i = %d, j = %d\n", r, i, j);

    return 0;
}
```

执行结果如下

```sh
$ ./main a
r = 0, i = 0, j = 0
$ ./main 1
r = 1, i = 1, j = 0
$ ./main 1:2
r = 2, i = 1, j = 2
```


### 溢出问题 {#溢出问题}

sscanf 最为人诟病的地方，是很容易出现缓冲区溢出错误。实际上 sscanf 是可以避免出现缓冲区溢出的，只要在书写任何字符串解析的格式时，注意加上其缓冲区尺寸的限制。如

```c
#include <stdio.h>

int main()
{
    char match[8] = "12345678"; // fill with values to test with terminating
                                // null character
    int len = sizeof(match)-1;
    const char *input = "hello world";
    char format[16];

    snprintf(format, sizeof(format), "%%%d[a-z ]", len);
    printf("format = %s\n", format);
    sscanf(input, format, match);
    printf("match = %s\n", match);
}
```

输出为

```sh
format = %7[a-z ]
match = hello w
```

这里 format 里面的 `%7` 就限定了拷贝过来的字符串长度为 7 ，也就是 match 的总长减一。 **最后一个字符留给字符串的结尾字符 `\0` ，sscanf 会自动添加。**

