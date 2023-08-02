---
layout: post
title: systemtap使用
categories: Blog
keywords: systemtap
---

## 安装

- 在centos下安装

```shell
yum install kernel-devel
yum install systemtap
yum install yum-utils				# 安装debuginfo工具
debuginfo-install glibc				# 安装glibc-debuginfo
debuginfo-install kernel			# 安装kernel-debuginfo
```

- 测试安装

```shell
stap -V
stap -L 'kernel.function("printk")'		# 测试kernel-debuginfo
stap -L 'process("/lib64/libc.so.6").function("malloc")'	# 测试glibc-debuginfo
```

- 一些报错

1. 安装debuginfo包

debuginfo-install安装kernel的debuginfo时，没找到debuginfo包时，

先找到内核构建的详细信息

```shell
rpm -qi kernel-devel
```

![My helpful screenshot]({{"/images/posts/blog/systemtap/2.png" | absolute_url}}) 

然后去对应的官网上找kernel-debuginfo和kernel-debuginfo-common包

2. 测试systemtap时报如下错

```shell
stap -ve 'probe begin {print("hello world\n)exit()"}'
```

错误如下

![My helpful screenshot]({{"/images/posts/blog/systemtap/1.png" | absolute_url}}) 

这是因为没有安装 kernel-devel包。通过yum安装

```shell
yum install kernel-devel
```



## 常规使用

- **查看内核函数**

如果查找的内核函数位于某个模块里

1. 查看fuse模块所有使用的函数

```shell
stap -L 'module("fuse").function("*")'
```

![My helpful screenshot]({{"/images/posts/blog/systemtap/3.png" | absolute_url}}) 

2. 直接看ceph模块的某个函数

```shell
stap -L 'module("fuse").function("ceph_write_inode")'
```

![My helpful screenshot]({{"/images/posts/blog/systemtap/4.png" | absolute_url}}) 

- **定位内核态函数位置**

比较典型的是定位内核系统调用函数在哪个文件上，可以通过grep查找，非常麻烦而且很慢。比如要找open系统调用在内核的实现，如果用SystemTap的话一个命令就搞定了：

```shell
stap -l 'kernel.function("sys_open")' 
```

![My helpful screenshot]({{"/images/posts/blog/systemtap/5.png" | absolute_url}}) 

- **设置探测点**

用stap的-L参数看能在哪些行上设probe

```
root@node ~/systemtap# stap -L 'kernel.statement("copy_process@/***/kernel/fork.c:*")' 
kernel.statement("copy_process@/***/kernel/fork.c:1134") 
...
kernel.statement("copy_process@/***/kernel/fork.c:1172") $clone_flags:long unsigned int $stack_start:long unsigned int $stack_size:long unsigned int $child_tidptr:int* $pid:struct pid* $trace:int $p:struct task_struct*
```

像GDB一样，SystemTap可以在vfs_read设置个probe，SystemTap代码如下：

```shell
probe kernel.statement("vfs_read@/***/fs/read_write.c:392")
{
    if ($count == 2334) { //过滤需要的信息
        printf("open: %s, read: %s\n", symname($file->f_op->open), symname($file->f_op->read));
    }
}
```



- **打印内核堆栈**

比如想打印iscsit_start_nopin_timer的调用过程

1, 先看iscsit_start_nopin_timer是哪个模块的

在dynamic_debug下的control中查看对应函数在哪个模块

2, 查看是否支持

```shell
stap -L 'module("iscsi_target_mod").function(""iscsit_start_nopin_timer")'
```

3, 写脚本test.stp

```shell
probe module("iscsi_target_mod").function(""iscsit_start_nopin_timer) {
	printf ("%d\n", __indent_timestamp())	    # 打印时间
	print_backtrace();
}
```

4, 执行

```
stap -v test.stp
```

查看调用栈不全，就需要加上参数

如果kernel的参数不全，就加-d kernel

如果其他模块不全，比如iscsi_tcp，就加-d iscsi_tcp，或者使用--all-modules方法强制将所有模块符号加载起来



- **打印变量**

1，查看该函数是否支持

```shell
stap -L 'module("libiscsi).function("iscsi_conn_stat")'
```

2，查看该函数中支持的变量查看

```shell
stap -L 'module("libiscsi").statement("iscsi_conn_start@drivers/scsi/libiscsi.c:*")' # 找到目标的变量位置。
```

3，写脚本查看变量

```shell
probe module("libiscsi").statement("iscsi_conn_start@drivers/scsi/libiscsi.c:2977") {
	printf("timeout:%d\n", $conn->timeout);
}
```

4，获取内核字符串

```shell
kernel_string(address)
```



- **用户态**

1, 查看二进制文件的所有函数

```shell
stap -L 'process("/sbin/iscsid").function("*")'
```

2, 查看用户态的参数

```shell
stap -L 'process("***").statement("*")'
```

3, 向下打印函数走向

​	使用官网上的call graph脚本

4, 打印时间

```shell
printf(\n%s\n", ctime(gettimeofday_s()))
```





## 例子

- **打印子进程名**

process.stp

```shell
probe kprocess.exec_complete {
  if (execname() == "find")
        printf("name: %s, %s pid: %d, tid: %d\n", name, execname(), pid(), tid());
}
probe kprocess.exec_complete {
  if (execname() == "io")
        printf("name: %s, %s pid: %d, tid: %d\n", name, execname(), pid(), tid());
}
```

