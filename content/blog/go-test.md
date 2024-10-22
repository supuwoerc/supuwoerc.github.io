---
title: Golang中的单元测试和模糊测试
date: 2024-10-22
authors:
  - name: Idris
    link: https://github.com/supuwoerc
excludeSearch: true
draft: true
---

随着项目的复杂度和层级越来越复杂，单元测试和模糊测试成为项目不可或缺的一部分，
本文介绍在Golang的项目中如何书写单元/模糊测试以及一些常见的测试技巧，帮助开发者发现潜在的问题。
<!--more-->

## 快速入门

Go 自带了一个轻量级的测试框架，由 go test 命令和 testing 包组成，官方推荐将测试文件和需要测试的文件放在同一个目录下：例如文件 add.go 文件中的方法 add 需要测试，
直接在同目录创建 add_test.go 即可。

所有结尾为 _test.go 的文件都被视为单元测试文件，不会被编译到可执行文件中，同时单元测试文件中的测试用例需要以 Test/Benchmark/Fuzz/Example 开头，下面是不同前缀的对应关系：

| 类型   | 函数前缀      | 作用              |
|------|-----------|-----------------|
| 基础测试 | Test      | 对需要测试的方法和业务进行测试 |
| 基准测试 | Benchmark | 测试方法的性能         |
| 模糊测试 | Fuzz      | 自动化测试方法         |
| 示例   | Example   | 自动化测试方法         |

下面简单介绍如何快速实现一个方法的测试：
```go {filename="add.go"}
// 需要测试的方法
func Add(a, b int) int {
    return a + b
}
```
创建单元测试文件 add_test.go 如下：
```go {filename="add_test.go"}
func TestAdd(t *testing.T) {
	if result := Add(1, 2); result != 3 {
		t.Errorf("Add(1, 2) = %d; want 3", result)
	}
}
```

执行 `go test` 会运行全部的测试用例，加入当前 package 下的全部测试，加入当前的包下只有这一个测试用例，输出如下：
```shell
PASS
ok      gin-web/test    0.438s
```

如果需要查看执行的详情，执行 `go test -v`即可：
```shell
go test -v
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
PASS
ok      gin-web/test    0.247s
```

如果需要查看测试的覆盖率，则加上 -cover 即可：
```shell
go test -cover
PASS
coverage: 100.0% of statements
ok      gin-web/test    0.253s
```

如果存在很多个测试用例，而你只希望执行某些测试，则加上 -run ，run 参数是一个正则匹配，你可以使用通配符之类的匹配规则：
```shell
go test -run TestAdd -v
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
PASS
ok      gin-web/test    0.253s
```
## testing包

这部分主要用于简单的介绍 testing 包下的一些常见成员及其常用的方法，部分内容会在下面的章节详细介绍。

### testing.T

* Error 和 Errorf：记录测试中的错误信息，同时标记测试为失败，不会停止测试函数的执行。

  * t.Error(args...)：类似于 fmt.Println，记录错误信息。
  
  * t.Errorf(format, args...)：类似于 fmt.Printf，使用格式化字符串记录错误信息。

* Fatal 和 Fatalf：记录测试中的错误信息，并立即终止当前测试函数的执行。

  * t.Fatal(args...)：类似于 fmt.Println，记录错误信息并终止测试。
  
  * t.Fatalf(format, args...)：类似于 fmt.Printf，使用格式化字符串记录错误信息并终止测试。

* Log 和 Logf：记录测试中的日志信息，但不影响测试的状态（不会标记为失败）。

  * t.Log(args...)：类似于 fmt.Println，记录日志信息。
  
  * t.Logf(format, args...)：类似于 fmt.Printf，使用格式化字符串记录日志信息。

* Run：用于运行一个子测试（subtest）。Run 方法可以创建一个逻辑上的子测试，它有自己独立的 *testing.T，并且可以并发执行。

  * t.Run(name string, f func(t *testing.T))：创建并执行一个子测试。

* Skip 和 Skipf：用于在特定条件下跳过测试。
  
  * t.Skip(args...)：类似于 fmt.Println，记录跳过的原因并跳过测试。
  
  * t.Skipf(format, args...)：类似于 fmt.Printf，使用格式化字符串记录跳过的原因并跳过测试。

* Helper：将当前函数标记为一个帮助函数，以便在记录错误时省略它的调用信息。这有助于提高错误日志的清晰度。
  
  * t.Helper()：在帮助函数的开头调用。

* Parallel：标记当前测试为可以并行运行，与其他并行测试一起执行。通常用于需要并发测试的场景。

  * t.Parallel()：在测试函数的开头调用。

* Cleanup：用于注册一个在测试完成后（无论成功还是失败）都会执行的清理函数。

    

### testing.B

* b.ResetTimer()：重置计时器，忽略之前的代码执行时间。例如，在基准测试中执行一些初始化操作后，重置计时器只记录实际的测试部分时间。

* b.StartTimer() 和 b.StopTimer()：控制计时器的启动和停止，适合需要手动控制基准测试时间的场景。

* b.RunParallel()：用于测试并行执行的性能。这个方法允许基准测试并行运行多个 Goroutine，从而测试并发性能。

### testing.M

testing.M 是 testing 包中用于控制整个测试过程的结构体。在需要进行测试前后全局初始化和清理操作时，testing.M 非常有用。它通常在 TestMain 函数中使用。

* TestMain：是一个特殊的测试主入口函数，签名为 func TestMain(m *testing.M)，它在所有测试开始之前执行，通常用于设置全局状态或初始化资源（例如数据库连接、读取配置等）。

  * m.Run() 方法用于运行所有测试，返回值是测试结果的状态码（0 表示成功）。

## 子测试
## 基准测试
## Setup & TearDown
## 模糊测试
