---
title: Go单元测试
date: 2017-06-23 17:33:44
tags: golang
categories: Golang
---

# 为什么需要单元测试

 开发程序其中很重要的一点是测试，我们如何保证代码的质量，如何保证每个函数是可运行，运行结果是正确的，又如何保证写出来的代码性能是好的，我们知道单元测试的重点在于发现程序设计或实现的逻辑错误，使问题及早暴露，便于问题的定位解决。对一个包做（单元）测试，需要写一些可以频繁（每次更新后）执行的小块测试单元来检查代码的正确性。于是我们必须写一些 Go 源文件来测试代码。测**试程序必须属于被测试的包**，并且文件名满足这种形式 `*_test.go`，所以测试代码和包中的业务代码是分开的。

 Go语言中自带有一个轻量级的测试框架`testing`和自带的`go test`命令来实现单元测试和性能测试，`testing`框架和其他语言中的测试框架类似，你可以基于这个框架写针对相应函数的测试用例。

`_test` 程序不会被普通的 Go 编译器编译，所以当放应用部署到生产环境时它们不会被部署；只有 gotest 会编译所有的程序：普通程序和测试程序。

另外建议安装[gotests](https://github.com/cweill/gotests)插件自动生成测试代码:

```
go get -u -v github.com/cweill/gotests/...
```

go单元测试的命令是

`go test`

go test 只会输出错误的信息，想要看详细的信息使用`go test -v`

# Go的自带单元测试的编写

## 1、如何编写测试用例

接下来我们在该目录下面创建两个文件：even.go和oddeven_test.go
**even.go:**

```
/*
用来测试前 100 个整数是否是偶数。这个函数属于 even 包。
*/
package even
func Even(i int) bool {
	return i%2 == 0
}
func Odd(i int) bool {
	return i%2 != 0
}
```

**oddeven_test.go:这是我们的单元测试文件，**

### **1.1测试文件的规则：**

1. 文件名必须是`_test.go`结尾的，这样在执行`go test`的时候才会执行到相应的代码
2. 你必须import `testing`这个包
3. 所有的测试用例函数必须是`Test`开头
4. 测试用例会按照源代码中写的顺序依次执行
5. 测试函数`TestXxx()`的参数是`testing.T`，我们可以使用该类型来记录错误或者是测试状态
6. 测试格式：`func TestXxx (t *testing.T)`,`Xxx`部分可以为任意的字母数字的组合，但是首字母不能是小写字母[a-z]，例如`Testintdiv`是错误的函数名。
7. 函数中通过调用`testing.T`的`Error`, `Errorf`, `FailNow`, `Fatal`, `FatalIf`方法，说明测试不通过，调用`Log`方法用来记录测试的信息。

**oddeven_test.go**

```go
package even
import "testing"
func TestEven(t *testing.T) {
	if !Even(10) { //!Even(10)==false,不会向下执行
		t.Log("10 must be even")
		t.Fail()
	}
	if Even(7) {
		t.Log("7 is not even")
		t.Fatal()
	}
	if Even(10) { //Even(10)==true,但是为了测试，让它执行t.Log()和 t.Fail()
		t.Log("Everything OK: 10 is even, just a test to see failed output!")
		t.Fail()
	}
}
func TestOdd(t *testing.T) {
	if !Odd(11) {
		t.Log(" 11 must be odd!")
		t.Fail()
	}
	if Odd(10) {
		t.Log(" 10 is not odd!")
		t.Fail()
	}
}
```

**使用go test 运行oddeven_test.go,只会出现错误信息**

> — FAIL: TestEven (0.00s)
>
> ```
> oddeven_test.go:16: Everything OK: 10 is even, just a test to see failed output!
> ```
>
> FAIL
> exit status 1
> FAIL MyStudy/GolangTest/even 0.364s

**使用go test -v，会将详细的信息都打印出来，通过的会有pass标识**

> === RUN TestEven
> — FAIL: TestEven (0.00s)
>
> ```
> oddeven_test.go:16: Everything OK: 10 is even, just a test to see failed output!
> ```
>
> === RUN TestOdd
> — PASS: TestOdd (0.00s)
> FAIL
> exit status 1
> FAIL MyStudy/GolangTest/even 0.397s

### 1.2单元测试的一些通知失败的方法

1）`func (t *T) Fail()`

```
标记测试函数为失败，然后继续执行（剩下的测试）。
```

2）`func (t *T) FailNow()`

```
标记测试函数为失败并中止执行；文件中别的测试也被略过，继续执行下一个文件。
```

3）`func (t *T) Log(args ...interface{})`

```
args 被用默认的格式格式化并打印到错误日志中。
```

4）`func (t *T) Fatal(args ...interface{})`

```
结合 先执行 3），然后执行 2）的效果。
```

# GoGonvey框架写单元测试

## 1、GoGonvey框架介绍

 Go 语言虽然自带单元测试功能，在 GoConvey 诞生之前也出现了许多第三方辅助库。但没有一个辅助库能够像 GoConvey 这样优雅地书写代码的单元测试，简洁的语法和舒适的界面能够让一个不爱书写单元测试的开发人员从此爱上单元测试。

### 1.1、GoGonvey的优点

1. GoConvey支持Go的本机[`testing`](http://golang.org/pkg/testing/)包。无论是网页界面还是DSL都不需要; 你可以独立使用任何一个。
2. GoConvey集成后`**go test**`，您可以[在终端中](https://github.com/smartystreets/goconvey/wiki/Execution)继续运行测试[，](https://github.com/smartystreets/goconvey/wiki/Execution)或者使用自动更新Web UI进行测试。
3. GoConvey还有web UI可以来方便查看代码覆盖率，以及单元测试错误信息

### 1.2、安装GoGonvey

```
go get github.com/smartystreets/goconvey
```

### 1.3、快速开始一个例子

写一个oddeven_goconvey_test.go 文件,将test.go改写成convey形式的

```go
package even

import (
	. "github.com/smartystreets/goconvey/convey"
	"testing"
)

func TestEvenConvey(t *testing.T) {
	Convey("Given some integer with a starting value", t, func() {
		Convey("When the integer is even", func() { //第一个Convey需要t参数，以后的不需要了
			Convey("10 must be even", func() {
				b := Even(10)
				So(b, ShouldBeTrue)
			})

			Convey("7 is not even", func() {
				b := Even(7)
				So(b, ShouldBeFalse)
			})

			Convey("Everything OK: 10 is even, just a test to see failed output!", func() {
				b := Even(10)
				So(b, ShouldBeFalse)
			})

		})
	})
}
```

 使用 GoConvey 书写单元测试，每个测试用例需要使用Convey函数包裹起来。它接受的第一个参数为 string 类型的描述；第二个参数一般为*testing.T，即本例中的变量 t；第三个参数为不接收任何参数也不返回任何值的函数（习惯以闭包的形式书写）。

Convey语句同样可以无限嵌套，以体现各个测试用例之间的关系，例如TestEvenConvey函数就采用了嵌套的方式体现它们之间的关系。需要注意的是，**只有最外层的Convey需要传入变量 t，内层的嵌套均不需要传入**。

最后，需要使用So语句来对条件进行判断。在本例中，我们只使用了 2 个不同类型的条件判断：ShouldBeTrue和ShouldBeFalse，分别表示值应该为 true、值应该false。

常用的有

1. `ShouldEqual`: 表示值应该想等
2. `ShouldResemble`: 进行深度相同检查，要有两个值`So(b, ShouldResemble, true)`
3. `ShouldBeTrue`: 表示值应该为true
4. `ShouldBeZeroValue`:表示值应该为0
5. `ShouldNotContainSubstring`:接收2字符串参数并确保第一个不包含第二个字符串。
6. `ShouldPanic`: 表示值应该panic

有关详细的条件列表，可以参见[官方文档](https://github.com/smartystreets/goconvey/wiki/Assertions)。

### 1.4、运行测试

#### 1.4.1、go test -v运行

现在，可以打开命令行，然后输入`go test -v`来进行测试。由于 GoConvey 兼容 Go 原生的单元测试，因此我们可以直接使用 Go 的命令来执行测试。 

> === RUN TestEvenConvey
>
> Given some integer with a starting value
> When the integer is even
> 10 must be even .
> 7 is not even .
> Everything OK: 10 is even, just a test to see failed output! x
>
> Failures: 
>
> D:/gopath/src/MyStudy/GolangTest/even/oddeven_goconvey_test.go
>
> Line 23:
>
> ```
> Expected: false
> Actual:   true
> ```
>
>  3 total assertions
>
> — FAIL: TestEvenConvey (0.00s)
> FAIL
> exit status 1
> FAIL MyStudy/GolangTest/even 0.401s

可以看到给出的信息比go自带的测试包多很多信息

#### 1.4.2、goconvey 运行测试

web界面查看数据

GoConvey 不仅支持在命令行进行人工调用调试命令，还有非常舒适的 Web 界面提供给开发者来进行自动化的编译测试工作。在项目目录下执行`goconvey` 同时可以打开浏览器查看localhost:8080

![](/home/zhaohq/blog/hexo/source/Go单元测试/带有错误的convey.png)

解析上面图片

### 1.5、GoConvey的一些功能

#### 测试自动运行

 只需保存`.go`![文件或单击图标执行测试](/home/zhaohq/blog/hexo/source/Go单元测试/更新.png)。您的浏览器页面将自动更新。

#### 覆盖报告

可以点击左上角的软件包名称来查看[Go的详细HTML覆盖率报告](http://blog.golang.org/cover#TOC_5.)。

#### 黑暗，光和自定义主题

GoConvey内置了两个主题。（在此页面上[切换主题](javascript:)以尝试！）您还可以使用第三方主题或自己滚动。

#### 暂停，恢复和审查

可以暂停自动测试执行，并查看最近的测试历史，以查看代码在何处，何时以及为什么中断。

在 Web 界面中，您可以设置界面主题，查看完整的测试结果，使用浏览器提醒等实用功能。

#### 其它功能：

1. 自动检测代码变动并编译测试
2. 半自动化书写测试用例：<http://localhost:8080/composer.html>
3. 查看测试覆盖率：<http://localhost:8080/reports/>
4. 临时屏蔽某个包的编译测试