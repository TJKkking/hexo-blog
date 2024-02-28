---
title: Docker镜像构建最佳实践
date: 2023-12-04 14:53:20
tags:
- Cloud
---

## 多阶段构建
示例程序：
```shell
/* hello.c */
int main () {
  puts("Hello, world!");
  return 0;
}
```
通过下面的 `Dockerfile` 构建镜像：
```shell
FROM gcc
COPY hello.c .
RUN gcc -o hello hello.c
CMD ["./hello"]
```
最后构建成功的镜像体积远远超过了 1 GB。因为该镜像包含了整个 `gcc `镜像的内容。
如果使用 `Ubuntu` 镜像，安装 C 编译器，最后编译程序，你会得到一个大概 300 MB 大小的镜像，比上面的镜像小多了。
但还是不够小，因为编译好的可执行文件还不到 20 KB：
```shell
$ ls -l hello
-rwxr-xr-x   1 root root 16384 Nov 18 14:36 hello
```
使用`Golang`的话也是类似的情况：
```shell
package main

import "fmt"

func main () {
  fmt.Println("Hello, world!")
}
```
使用基础镜像 `golang` 构建的镜像大小是 800 MB，而编译后的可执行文件只有 2 MB 大小：
```shell
$ ls -l hello
-rwxr-xr-x 1 root root 2008801 Jan 15 16:41 hello
```

**多阶段构建**的核心思想是：我不想在最终的镜像中包含一堆 中间过程需要的C 或 Go 编译器和整个编译工具链，我只要一个编译好的可执行文件。
使用`FROM`进行多阶段构建的典型例子：
```shell
FROM gcc AS mybuildstage
COPY hello.c .
RUN gcc -o hello hello.c
FROM ubuntu
COPY --from=mybuildstage hello .
CMD ["./hello"]
```
解释：以上的Dockerfile分为两个阶段。首先使用基础镜像 `gcc` 来编译程序 `hello.c`，此阶段命名为`mybuildstage`。然后启动一个新的构建阶段，以 `ubuntu` 作为基础镜像，将可执行文件 hello 从上一阶段`COPY`到最终的镜像中并运行。
:::info
Tips：在声明构建阶段时，可以不必使用关键词 `AS`，最终阶段拷贝文件时可以直接使用序号表示之前的构建阶段（从零开始）。
如果 `Dockerfile` 内容不是很复杂，构建阶段也不是很多，可以直接使用序号表示构建阶段。一旦 `Dockerfile` 构建阶段增多，最好还是通过关键词 `AS` 为每个阶段命名，这样也便于后期维护。
:::
```
COPY --from=mybuildstage hello .
COPY --from=0 hello .
```
对比镜像：
```shell
$ docker images minimage
REPOSITORY          TAG                    ...         SIZE
minimage            hello-c.gcc            ...         1.14GB
minimage            hello-c.gcc.ubuntu     ...         64.2MB
```
最终的镜像大小是 64 MB，比之前的 1.1 GB 减少了 95%，效果显著。
## 基础镜像
有很多官方提供的基础镜像已经经过了足够的优化，建议使用。比如`CentOS`，`Debian`，`Fedora` 和以及`Ubuntu`等。
更小的还有 `scratch` 或者 `busybox`：
关于 `scratch`：

- 一个空镜像，只能用于构建镜像，通过 FROM scratch
- 在构建一些基础镜像，比如 debian 、 busybox，非常有用
- 用于构建超少镜像，比如构建一个包含所有库的[二进制文件](https://www.zhihu.com/search?q=%E4%BA%8C%E8%BF%9B%E5%88%B6%E6%96%87%E4%BB%B6&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22article%22%2C%22sourceId%22%3A%22430759892%22%7D)

关于 `busybox`

- 只有 1~5M 的大小
- 包含了常用的 UNIX 工具
- 非常方便构建小镜像

这些超小的基础镜像，结合能生成静态原生 ELF 文件的编译语言，比如C/C++和Golang，可以特别方便构建超小的镜像。
