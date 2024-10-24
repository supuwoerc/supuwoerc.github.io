---
title: Golang中的mock
date: 2024-10-24
authors:
- name: Idris
link: https://github.com/supuwoerc
excludeSearch: true
draft: true
tags:
  - Golang
  - 测试
---

gomock 是 Go 编程语言的模拟框架， 它与 Go 的内置测试包很好地集成在一起，但也可以在其他上下文中使用，本文介绍 gomock 的使用场景和方法。

<!--more-->

<!--more-->
{{< callout type="warning" >}}
2023年6月官方停止维护 gomock
{{< /callout >}}

> 可惜的是 2023年6月官方停止维护了，但是我们依旧可以使用 uber 团队维护的分支，gomock 依旧是 go 测试中非常值得学习的框架。

## 介绍

gomock 主要的作用是帮助我们模拟复杂的函数和对象，例如外部接口/RPC调用/数据库连接/文件操作，当然，除了 gomock 社区还有非常多的其他 mock 框架，例如针对数据库这类的复杂对象进行 mock ，总的来说mock类型的框架目的就是
为了帮助我们模拟复杂对象的行为，减少我们手动创建这些模拟对象的额外负担。

配合 gomock 的还有 mockgen 工具，用来辅助我们生成测试代码，下面介绍如何安装和使用它们。

## 安装

```shell
go install github.com/golang/mock/mockgen@v1.6.0
```
安装完成后终端验证 `mockgen -version` 输出如下：

```shell
mockgen -version
v1.6.0
```

## mockgen

安装完成 mockgen 之后，我们可以简单了解一下它支持那些操作，以及常用的 option，终端输入 `mockgen`了解相关的信息，下面我添加了翻译来方便理解：

```shell
mockgen has two modes of operation: source and reflect.
// mockgen 有两种运行模式：源模式和反射模式
Source mode generates mock interfaces from a source file.
It is enabled by using the -source flag. Other flags that
may be useful in this mode are -imports and -aux_files.
Example:
        mockgen -source=foo.go [other options]
// 源模式是从源文件中生成 mock 接口，使用 -source 来启用源模式，除此之外还可以配合使用 -imports 和 -aux_files。
Reflect mode generates mock interfaces by building a program
that uses reflection to understand interfaces. It is enabled
by passing two non-flag arguments: an import path, and a
comma-separated list of symbols.
Example:
        mockgen database/sql/driver Conn,Driver

  -aux_files string
        (source mode) Comma-separated pkg=path pairs of auxiliary Go source files.
  -build_flags string
        (reflect mode) Additional flags for go build.
  -copyright_file string
        Copyright file used to add copyright header
  -debug_parser
        Print out parser results only.
  -destination string
        Output file; defaults to stdout.
  -exec_only string
        (reflect mode) If set, execute this reflection program.
  -imports string
        (source mode) Comma-separated name=path pairs of explicit imports to use.
  -mock_names string
        Comma-separated interfaceName=mockName pairs of explicit mock names to use. Mock names default to 'Mock'+ interfaceName suffix.
  -package string
        Package of the generated code; defaults to the package of the input with a 'mock_' prefix.
  -prog_only
        (reflect mode) Only generate the reflection program; write it to stdout and exit.
  -self_package string
        The full package import path for the generated code. The purpose of this flag is to prevent import cycles in the generated code by trying to include its own package. This can happen if the mock's package is set to one of its inputs (usually the main one) and the output is stdio so mockgen cannot detect the final output package. Setting this flag will then tell mockgen which import to exclude.
  -source string
        (source mode) Input Go source file; enables source mode.
  -version
        Print version.
  -write_package_comment
        Writes package documentation comment (godoc) if true. (default true)
2024/10/24 12:00:01 Expected exactly two arguments
```
