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

### 跳过部分测试

为了节省时间，我们借助 testing.Short 和 testing.T.Skip 方法来标记和跳过当前的测试用例，当运行的实施传入 -short 参数，就可以跳过一部分测试代码。

```go
func TestShort(t *testing.T) {
	if testing.Short() {
		t.Skip("skipping test in short mode.")
	}
	t.Log("代码逻辑")
}
```
在命令行中执行 ` go test -short -v` 后，将跳过当前测试用例：

```shell
=== RUN   TestShort
    a_test.go:9: skipping test in short mode.
--- SKIP: TestShort (0.00s)
PASS
ok      gin-web 0.314s
```

## 子测试 & Table-Driven Approach

前面介绍了 *testing.T 的 Run 方法，支持创建子测试，我们为前问的 Add 方法来创建多个子测试。

```go
func TestAdd(t *testing.T) {
	t.Run("1+1", func(t *testing.T) {
		if got := Add(1, 1); got != 2 {
			t.Errorf("Add(1,1) = %d, want %d\n", got, 2)
		}
	})
	t.Run("-1+1", func(t *testing.T) {
		if got := Add(-1, 1); got != 0 {
			t.Errorf("Add(-1,1) = %d, want %d\n", got, 0)
		}
	})
	t.Run("-1+-1", func(t *testing.T) {
		if got := Add(-1, -1); got != -2 {
			t.Errorf("Add(-1,-1) = %d, want %d\n", got, -2)
		}
	})
}
```
执行 `go test -v`后，我们能查看到每一个子用例的执行结果：
```shell
=== RUN   TestAdd
=== RUN   TestAdd/1+1
=== RUN   TestAdd/-1+1
=== RUN   TestAdd/-1+-1
--- PASS: TestAdd (0.00s)
    --- PASS: TestAdd/1+1 (0.00s)
    --- PASS: TestAdd/-1+1 (0.00s)
    --- PASS: TestAdd/-1+-1 (0.00s)
PASS
ok      gin-web 0.293s
```

使用子测试可以为一个测试目标创建多个用例，但是依旧比较麻烦，所以官方还推荐我们使用 Table-Driven Approach 来实现方法的大量数据集测试。

Table-Driven Approach 也就是为测试用例准备表格一样的数据来作为参数，不需要调整测试用例本身，再扩展用例的时候只需要扩展这份数据即可，例如下面这个例子来测试前问的 Add 方法：

```go
func TestAdd(t *testing.T) {
	cases := []struct {
		a, b, expect int64
	}{
		{1, 1, 2},
		{1, -1, 0},
		{-1, -1, -2},
	}
	for _, item := range cases {
		if got := Add(item.a, item.b); got != item.expect {
			t.Errorf("Add(%d, %d) = %d, want %d", item.a, item.b, got, item.expect)
		}
	}
}
```
在后续的扩展中，我们只需要扩展 cases 即可。

## 并行测试

前面提到的子测试和 Table-Driven Approach 存在大量的测试用例需要执行，这些用例的执行是顺序的，我们很容易发现我们可以借助 Golang 优秀的并发实现来完成用例的并发执行。

例如存在这样的三个测试：
```go
func TestA(t *testing.T) {
	time.Sleep(time.Second)
	t.Log("TestA is running...")
}

func TestB(t *testing.T) {
	time.Sleep(3 * time.Second)
	t.Log("TestB is running...")
}

func TestC(t *testing.T) {
	t.Run("TestC1", func(t *testing.T) {
		time.Sleep(3 * time.Second)
		t.Log("TestC1 is running...")
	})
	t.Run("TestC2", func(t *testing.T) {
		time.Sleep(5 * time.Second)
		t.Log("TestC2 is running...")
	})
}
```
执行 `go test -v` 查看运行结果和耗时，发现一共需要 12s 以上，如果存在很多耗时的测试用例，这样的时间是无法忍受的：
```shell
=== RUN   TestA
    a_test.go:32: TestA is running...
--- PASS: TestA (1.00s)
=== RUN   TestB
    a_test.go:37: TestB is running...
--- PASS: TestB (3.00s)
=== RUN   TestC
=== RUN   TestC/TestC1
    a_test.go:43: TestC1 is running...
=== RUN   TestC/TestC2
    a_test.go:47: TestC2 is running...
--- PASS: TestC (8.00s)
    --- PASS: TestC/TestC1 (3.00s)
    --- PASS: TestC/TestC2 (5.00s)
PASS
ok      gin-web 12.327s
```
下面我们借助 `Parallel` 来开启并发执行用例：

```go
func TestA(t *testing.T) {
	t.Parallel()
	time.Sleep(time.Second)
	t.Log("TestA is running...")
}

func TestB(t *testing.T) {
	t.Parallel()
	time.Sleep(3 * time.Second)
	t.Log("TestB is running...")
}

func TestC(t *testing.T) {
	t.Parallel()
	t.Run("TestC1", func(t *testing.T) {
		t.Parallel()
		time.Sleep(3 * time.Second)
		t.Log("TestC1 is running...")
	})
	t.Run("TestC2", func(t *testing.T) {
		t.Parallel()
		time.Sleep(5 * time.Second)
		t.Log("TestC2 is running...")
	})
}
```
再次执行 `go test -v` 查看执行，发现只需要 5s 左右了：

```shell
=== RUN   TestA
=== PAUSE TestA
=== RUN   TestB
=== PAUSE TestB
=== RUN   TestC
=== PAUSE TestC
=== CONT  TestA
=== CONT  TestB
=== CONT  TestC
=== RUN   TestC/TestC1
=== PAUSE TestC/TestC1
=== RUN   TestC/TestC2
=== PAUSE TestC/TestC2
=== CONT  TestC/TestC1
=== CONT  TestC/TestC2
=== NAME  TestA
    a_test.go:33: TestA is running...
--- PASS: TestA (1.00s)
=== NAME  TestC/TestC1
    a_test.go:47: TestC1 is running...
=== NAME  TestB
    a_test.go:39: TestB is running...
--- PASS: TestB (3.00s)
=== NAME  TestC/TestC2
    a_test.go:52: TestC2 is running...
--- PASS: TestC (0.00s)
    --- PASS: TestC/TestC1 (3.00s)
    --- PASS: TestC/TestC2 (5.00s)
PASS
ok      gin-web 5.315s
```
并行测试能在子测试场景下节约大量的执行时间，尤其在子测试很多同时子测试存在耗时操作的时候，效率提升明显。


## 帮助函数
## Setup & TearDown
## 网络测试
## 基准测试
## 模糊测试
## 示例函数Example
## 第三方库
### gotests

首先介绍：[cweill/gotests](https://github.com/cweill/gotests) ，这是一个为我们自动生成 Table-Driven Approach 用例的库。

#### 安装
```shell
$ go get -u github.com/cweill/gotests/...
```
#### 使用
```shell
$ gotests [options] PATH ...
```
### 参数选项
```shell
  -all                  为所有功能和方法生成测试

  -excl                 为不匹配的函数和方法生成测试。优先于 -only、-exported 和 -all

  -exported             为导出函数和方法生成测试。优先于 -only 和 -all

  -i                    在错误信息中打印测试输入

  -only                 为仅匹配的函数和方法生成测试。优先于 -all

  -nosubtests           当 >= Go 1.7 时禁用子测试生成

  -parallel             >= Go 1.7 时启用并行子测试生成。

  -w                    将输出写入（测试）文件，而不是 stdout

  -template_dir         包含自定义测试代码模板的目录路径。优先于 优先于 -template。也可通过环境变量 GOTESTS_TEMPLATE_DIR 设置。

  -template             指定自定义测试代码模板，如 testify。也可通过环境变量 GOTESTS_TEMPLATE 设置

  -template_params_file 通过 json 文件读取模板的外部参数

  -template_params      使用 stdin 通过 json 读取模板的外部参数
```
我要为 abc.go 这个文件里的方法创建测试用例：
 
```shell
gotests -all abc.go
```

执行完成后将生成对应的文件和测试用例，我们只需要补充 case 即可。