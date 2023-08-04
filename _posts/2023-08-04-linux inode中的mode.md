---
layout: post
title: inode中mode字段
categories: Blog
keywords: linux 文件系统
---

1，如何判断文件的类型：目录/文件
2，如何获取文件的用户权限
上面的问题如果要回答，得研究inode中的mode成员
在struct inode中有一个成员变量

```c
umode_t i_mode,
typedef unsigned short umode_t
```

umode_t是16位的2进制数，保存的就是文件类型及用户权限信息

![My helpful screenshot]({{"/images/posts/blog/mode/mode1.png" | absolute_url}}) 



## 文件类型位

只要知道了mode值就可以知道该文件的类型，即看mode的高0-3位的值

```c
#define         S_IFMT  0170000 	/* type of file ，文件类型掩码*/
#define         S_IFREG 0100000 	/* regular 普通文件*/
#define         S_IFBLK 0060000 	/* block special 块设备文件*/
#define         S_IFDIR 0040000 	/* directory 目录文件*/
#define         S_IFCHR 0020000 	/* character special 字符设备文件*/
......
```

S_IFMT是文件类型掩码，其值是0170000，转换成二进制就是1111 0000 0000 0000，
S_IFMT就是用来取其需要判断文件类型的mode的0--3位
所以判断文件还是目录的方法就出来了

```c
#define         S_ISDIR(m)      (((m) & S_IFMT) == S_IFDIR)    // 判断是否是目录文件
#define         S_ISREG(m)      (((m) & S_IFMT) == S_IFREG)    // 判断是否是普通文件
```



## SUID，SGID，sticky位

SUID 是 Set User ID, SGID 是 Set Group ID的意思。具体怎么用，暂时不研究

## 文件权限

文件权限分为属主权限、属主组权限和其他用户权限，即我们所知的rwxrwxrwx（777）之类

r 读权限read  4，即100表示有读权限

w 写权限write 2，即010表示有写权限

x 操作权限execute  1，即001表示有执行权限。


