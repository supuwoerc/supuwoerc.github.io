---
title: Golang中的mock
date: 2024-10-24
authors:
- name: Idris
link: https://github.com/supuwoerc
excludeSearch: true
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
go get -u github.com/golang/mock/gomock
go install github.com/golang/mock/mockgen@v1.6.0
```
安装完成后终端验证 `mockgen -version` 输出如下：

```shell
mockgen -version
v1.6.0
```

## mockgen

安装完成 mockgen 之后，我们可以简单了解一下它支持那些操作，以及常用的 option，终端输入 `mockgen`了解相关的信息：

* mockgen 有两种运行模式：源模式和反射模式

* 源模式是从源文件中生成 mock 接口，使用 -source 来启用源模式，除此之外还可以配合使用 -imports 和 -aux_files 等参数。

* 反射模式通过反射来实现一个接口的模拟，需要传入导入的路径和一个逗号分割的参数列表来创建。

以下是 Gomock 的一般使用步骤：

* 定义接口：首先，您需要定义一个要模拟的接口。

* 生成模拟代码：在包含接口定义的包目录下，运行 gomock 命令来生成模拟代码。例如：gomock --package=your_package_name
  * --package ：指定生成的模拟代码所在的包名。
  * --write_package_comment ：在生成的代码中添加包注释。
  * --self_package ：指定生成的模拟对象属于当前包。
  * --output ：指定生成模拟代码的输出文件路径。

## 代码示例
首先声明接口，在 main.go 文件中创建一个接口，同时借助 go:generate 来帮助我们自动生成相关的代码：

```go
//go:generate mockgen -destination=./mock_human.go -package=main -source=main.go
type Human interface {
	Say(str string) string
}
```
执行：`go generate ./...` 之后，会在当前文件夹下生成一个 mock_human.go 文件，下面我们编写测试文件 main_test.go，添加测试用例。

```go
func Test_mock_human(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()
	mockHuman := NewMockHuman(ctrl)
	mockHuman.EXPECT().Say("test").Return("test")
	if mockHuman.Say("test") != "test" {
		t.Errorf("say fail")
	}
}
```
执行测试，输出如下：

```shell
=== RUN   Test_mock_human
--- PASS: Test_mock_human (0.00s)
PASS
```
上面的例子我们仅仅是做演示，方法中用到的方法下面做一下说明，同时将其他常用的方法也做一下说明。

```go
func Test_mock_human(t *testing.T) {
	t.Run("指定方法的参数和返回值", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// 设置方法的入参和返回值
		mockHuman.EXPECT().Say("test").Return("test result")
		if mockHuman.Say("test") != "test result" {
			t.Errorf("say fail")
		}
	})
	t.Run("EXPECT断言方法被调用", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// EXPECT() 方法会断言方法被调用一次
		mockHuman.EXPECT().Say("run 1 times").Return("run 1 times result")
		// 如果不使用指定参数调用方法，会报错：aborting test due to missing call(s)
		if ret := mockHuman.Say("run 1 times"); ret != "run 1 times result" {
			t.Errorf("say fail:%v", ret)
		}
	})
	t.Run("指定方法的调用次数", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		mockHuman.EXPECT().Say("run 2 times").Return("run 2 times result").Times(2)
		if ret := mockHuman.Say("run 2 times"); ret != "run 2 times result" {
			t.Errorf("say fail:%v", ret)
		}
		if ret := mockHuman.Say("run 2 times"); ret != "run 2 times result" {
			t.Errorf("say fail:%v", ret)
		}
	})
	t.Run("不指定方法的调用次数", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// 可以调用mock方法任意此处(包含0次)
		mockHuman.EXPECT().Say("run any times").Return("run any times result").AnyTimes()
	})
	t.Run("指定方法的最少调用次数", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// 至少调用mock方法若干次
		mockHuman.EXPECT().Say("min times").Return("min times result").MinTimes(1)
		if ret := mockHuman.Say("min times"); ret != "min times result" {
			t.Errorf("say fail:%v", ret)
		}
	})
	t.Run("指定方法的最多调用次数", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// 至多调用mock方法若干次
		mockHuman.EXPECT().Say("max times").Return("max times result").MaxTimes(1)
		if ret := mockHuman.Say("max times"); ret != "max times result" {
			t.Errorf("say fail:%v", ret)
		}
	})
	t.Run("指定方法的实现细节-DoAndReturn", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// 覆盖方法的实现,自定义方法的mock实现
		mockHuman.EXPECT().Say("do and return").DoAndReturn(func(str string) string {
			return "custom return content"
		})
		if ret := mockHuman.Say("do and return"); ret != "custom return content" {
			t.Errorf("say fail:%v", ret)
		}
	})
	t.Run("指定方法的实现细节-Do", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// 覆盖方法的实现,自定义方法的mock实现
		mockHuman.EXPECT().Say("do and return").Do(func(s string) {
			fmt.Printf("重新实现方法,%s", s)
		}).Return("custom return content")
		if ret := mockHuman.Say("do and return"); ret != "custom return content" {
			t.Errorf("say fail:%v", ret)
		}
	})
	t.Run("不指定具体参数", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// 这里表示无论传入什么字符串
		mockHuman.EXPECT().Say(gomock.Any()).Return("any arg is okay2").AnyTimes()
		if ret := mockHuman.Say("1"); ret != "any arg is okay2" {
			t.Errorf("say fail:%v", ret)
		}
		if ret := mockHuman.Say("2"); ret != "any arg is okay2" {
			t.Errorf("say fail:%v", ret)
		}
	})
	t.Run("指定传入参数", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// 这里表示参数必须传入什么字符串
		mockHuman.EXPECT().Say(gomock.Eq("valid string")).Return("valid result")
		if ret := mockHuman.Say("valid string"); ret != "valid result" {
			t.Errorf("say fail:%v", ret)
		}
	})
	t.Run("指定传入参数不是某一个具体值", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// 这里表示传入的参数不能是 valid string
		mockHuman.EXPECT().Say(gomock.Not("valid string")).Return("valid result")
		if ret := mockHuman.Say("invalid string"); ret != "valid result" {
			t.Errorf("say fail:%v", ret)
		}
	})
	t.Run("指定传入参数是nil", func(t *testing.T) {
		ctrl := gomock.NewController(t)
		defer ctrl.Finish()
		mockHuman := NewMockHuman(ctrl)
		// 这里表示传入的参数必须是nil
		mockHuman.EXPECT().Say2(gomock.Nil()).Return("valid result")
		if ret := mockHuman.Say2(nil); ret != "valid result" {
			t.Errorf("say fail:%v", ret)
		}
	})
}
```
涉及到的方法和方法的作用：

* EXPECT：用于设置对模拟对象方法调用的期望，可以定义模拟方法在被调用时的各种期望行为，例如期望的调用次数、传入的参数、返回的值以及可能的副作用操作

* Times：用于指定期望的方法调用次数。

* AnyTimes：表示模拟的方法可以被调用任意次数，包括 0 次。

* MinTimes：指定模拟方法被调用的最小次数。

* MaxTimes：指定模拟方法被调用的最大次数。

* Do：模拟方法被调用时执行一个自定义的函数，这个函数可以包含一些额外的逻辑或副作用。

* DoAndReturn：与 Do 类似，但同时还能指定返回值。

* Eq：用于验证传入模拟方法的参数是否与指定的值相等。

* Not： 常与其他匹配器结合使用，用于表示相反的条件。

* Nil：用于检查某个值是否为 nil 。

以上就是 gomock 的使用方法和一些概念。