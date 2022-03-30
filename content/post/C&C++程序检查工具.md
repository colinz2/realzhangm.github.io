---
title: C/C++代码检查&运行时检查工具
description: C/C++代码检查&运行时检查工具
author: realzhangm
date: 2018-01-14
lastmod: 2018-01-14
tags: ["代码检查工具"]
---

## 静态代码检查工具

### scan-build

在源码目录下执行make

```
scan-build make
```
##### REF

http://clang-analyzer.llvm.org/

### infer

##### linux下安装

```
VERSION=0.XX.Y; \
curl -sSL "https://github.com/facebook/infer/releases/download/v$VERSION/infer-linux64-v$VERSION.tar.xz" \
| sudo tar -C /opt -xJ && \
ln -s "/opt/infer-linux64-v$VERSION/bin/infer" /usr/local/bin/infer
```
##### mac下安装

```
brew install infer
```
##### 使用

在源码目录下执行make

```
infer run -- make
```


## 运行时工具

### 谷歌sanitizers

- 谷歌sanitizers系列工具用于程序内存分析. (https://github.com/google/sanitizers)
  sanitizers以库的形式使用. GCC, Clang等编译器都已经集成, 编译程序时使用特定参数(-fsanitize=leak), 会将特定代码自动插入到程序中, 并链接sanitizers库.
  LeakSanitizer用于内存泄漏分析. GCC5.3以上支持LeakSanitizer.

- 例如使用gcc -fsanitize=leak 编译的程序. 通过SIGINT等信号退出程序, 分析结果会自动打印到终端

查看test进程内存使用大小

```
cat /proc/pidof test/status | grep VmRSS:
```

##### REF

http://fbinfer.com/docs/getting-started.html

https://github.com/google/sanitizers/wiki

## valgrind

```
 valgrind ./test  --tool=memcheck  --leak-check=full
```



##  热点函数检查

### Gperftools 

https://github.com/gperftools/gperftools

### Linux Perf工具

##### REF

https://perf.wiki.kernel.org/index.php/Main_Page

http://www.brendangregg.com/perf.html