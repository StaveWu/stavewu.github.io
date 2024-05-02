---
layout: post
title:  "spec依赖检查机制"
date:   2023-07-01 23:00:02 +0800
categories: 生态
tag: 兼容性
---

spec文件的依赖是怎么检查的？

编译阶段：通过BuildRequires；运行阶段：通过Requires

```bash
BuildRequires:  python
Requires:       python
```

具体检查方式：

查看rpmbuild源码：[Ftp - /releases/rpm-4.15.x/ :: Oregon State University Open Source Lab](http://ftp.rpm.org/releases/rpm-4.15.x/)

BuildRequires被解释为使用宏__spec_buildrequires_template：

![image-20230701170955227](/images/spec依赖检查机制/image-20230701170955227.png)

该宏具体内容可以通过rpm --eval展开：

```bash
rpm --eval %{__spec_buildrequires_template}
```

![image-20230701171040194](/images/spec依赖检查机制/image-20230701171040194.png)

这里涉及到一个环境变量PKG_CONFIG_PATH，使用了pkg-config工具，关于该工具介绍查看：[Guide to pkg-config (people.freedesktop.org)](https://people.freedesktop.org/~dbn/pkg-config-guide.html)

该工具的作用：

- 避免硬编码lib库路径：

```bash
gcc test.c `pkg-config --libs --cflags glib-2.0`

[root@openEuler2003SP1 SPECS]# pkg-config --libs --cflags glib-2.0
-I/usr/include/glib-2.0 -I/usr/lib64/glib-2.0/include -lglib-2.0
```

- 依赖版本检查

```bash
$ pkg-config --libs "bar >= 2.7"
Requested 'bar >= 2.7' but version of bar is 2.1.2
```

如果bar依赖foo，则使用以下命令将连带出完整的依赖链：

```bash
[root@openEuler2003SP1 test]# pkg-config --libs --static foo
-L/usr/lib -lfoo
[root@openEuler2003SP1 test]# pkg-config --libs --static bar
-L/usr/lib -lbar -L/usr/lib -lfoo
```

我们常见的configure输出的package not found错误，其实就是来自pkg-config的判断：

```bash
[root@openEuler2003SP1 test]# pkg-config --exists --print-errors xoxo
Package xoxo was not found in the pkg-config search path.
Perhaps you should add the directory containing `xoxo.pc'
to the PKG_CONFIG_PATH environment variable
Package 'xoxo', required by 'virtual:world', not found
```



我们再看下平时build阶段经常依赖的devel包，里面都有什么：

![image-20230701173341798](/images/spec依赖检查机制/image-20230701173341798.png)

可以看到有三部分，header、so、pc

header对应到头文件函数声明，so对应的函数具体实现，pc就是pkg-config的元数据文件了，这三元组最终组合形成完整的依赖编译命令：

```bash
gcc test.c -I/usr/include/foo -L/usr/lib -lbar -L/usr/lib -lfoo
```

现在回到spec，BuildRequires不满足打印示例：

```bash
[root@openEuler2003SP1 SPECS]# rpmbuild -ba daos.spec
error: Failed build dependencies:
        daos-raft-devel = 0.9.1-1401.gc18bcb8 is needed by daos-2.2.0-4.aarch64
        go >= 1.14 is needed by daos-2.2.0-4.aarch64
```

这些error依赖检查来自：

![image-20230701173925324](/images/spec依赖检查机制/image-20230701173925324.png)

![image-20230701174018570](/images/spec依赖检查机制/image-20230701174018570.png)

ts变量：transaction set，在rpmts.h中定义了一系列关于该变量的使用方法：

![image-20230701175121884](/images/spec依赖检查机制/image-20230701175121884.png)

所以rpm检查依赖的底层逻辑是通过查询数据库，数据库路径可以通过宏查看：

```bash
[root@openEuler2003SP1 SPECS]# rpm --eval "%{_db_backend}"
bdb
[root@openEuler2003SP1 SPECS]# rpm --eval %{_dbpath}
/var/lib/rpm
[root@openEuler2003SP1 SPECS]# ls /var/lib/rpm
Basenames     __db.001  __db.003  Enhancename      Group       Name          Packages     Recommendname  Sha1header  Suggestname     Transfiletriggername
Conflictname  __db.002  Dirnames  Filetriggername  Installtid  Obsoletename  Providename  Requirename    Sigmd5      Supplementname  Triggername
```

它是一个bdb数据库（旧的也有使用sqlite：[rpm.org - RPM Database Recovery](https://rpm.org/user_doc/db_recovery.html)）



devel包如何构建：[linux - Building both devel and normal version of a RPM package - Stack Overflow](https://stackoverflow.com/questions/2913130/building-both-devel-and-normal-version-of-a-rpm-package)

一般不会单独写一个devel的spec，而是和binary rpm的spec共用，通过在各个章节后面带devel标记区分：

```bash
%package devel
%description devel
%files devel
```

