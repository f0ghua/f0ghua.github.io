# C 语言常用库函数-strncpy


## 函数原型 {#函数原型}

```c
#include <string.h>

char *strncpy(char *dest, const char *src, size_t n);
```


## 用法分析 {#用法分析}

strncpy 由于添加了长度限制，避免了缓冲区溢出的问题。但在使用这个函数的时候必须要注意:

strncpy 只有在源字符串的长度小于参数 n 时，它才会用 NUL(或者'\\0')来结束字符串。

因此正确的使用 strncpy 的方法是，当拷贝源字符串的一部分时，使用 strncpy 之后，自己手工添加 NUL 来结束字符串。

```c
#define BUFSIZ 265
int main(int argc, char **argv)
{
    char buf[BUFSIZ];

    strncpy(buf, argv[1], sizeof(buf) - 1);
    buf[sizeof(buf) - 1] = '\0'; /* 防止buf没有初始化为0，结束字符串 */

    return 0;
}
```

这里可以看到，要正确使用 strncpy 还是挺麻烦的一件事情。


## 注意事项 {#注意事项}


### strlcpy {#strlcpy}

其实在 openbsd 里面提供了一个库函数 strlcpy 。它保证了目的字串总是以 NUL 结尾，并且返回值总是目的字串的全长(不包括 NUL)。其源码如下

```c
/*      $OpenBSD: strlcpy.c,v 1.11 2006/05/05 15:27:38 millert Exp $        */

/*
 * Copyright (c) 1998 Todd C. Miller <Todd.Miller@courtesan.com>
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
 * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
 * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
 * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
 * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
 * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 */

#include <sys/types.h>
#include <string.h>

/*
 * Copy src to string dst of size siz.  At most siz-1 characters
 * will be copied.  Always NUL terminates (unless siz == 0).
 * Returns strlen(src); if retval >= siz, truncation occurred.
 */
size_t
strlcpy(char *dst, const char *src, size_t siz)
{
    char *d = dst;
    const char *s = src;
    size_t n = siz;

    /* Copy as many bytes as will fit */
    if (n != 0) {
        while (--n != 0) {
            if ((*d++ = *s++) == '\0')
                break;
        }
    }

    /* Not enough room in dst, add NUL and traverse rest of src */
    if (n == 0) {
        if (siz != 0)
            *d = '\0';                /* NUL-terminate dst */
        while (*s++)
            ;
    }

    return(s - src - 1);        /* count does not include NUL */
}
```


### strncpy vs snprintf {#strncpy-vs-snprintf}

除了使用 strlcpy，还有个好的选择是使用 snprintf。

snprinf 可以完成 strncpy 的功能，而且比 strncpy 更有效率。看一下 glibc 中
strncpy 的代码

```c
char *
STRNCPY (char *s1, const char *s2, size_t n)
{
    size_t size = __strnlen (s2, n);
    if (size != n)
        memset (s1 + size, '\0', n - size);
    return memcpy (s1, s2, size);
}
```

我们可以看到，当 s2 的长度小于 n 的时候，strncpy 会把 s1 中从 s2 的长度以后的所有字节都设置成 `'\0'` ，显然，这个对于我们大部分情况下的需求来说是没有必要的，只要将字符串的最后一个字节置 NULL 即可。这也是 snprinf 的做法。

