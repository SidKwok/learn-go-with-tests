# 迭代

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/for)**

要在 Go 中重复做某件事，你需要 `for`。Go 中没有 `while`、`do`、`until` 这些关键字，你只能用 `for`。这是好事！

我们来给一个把字符重复 5 次的函数写测试。

到目前为止没什么新东西，所以试着自己写出来练练手。

## 先写测试

```go
package iteration

import "testing"

func TestRepeat(t *testing.T) {
	repeated := Repeat("a")
	expected := "aaaaa"

	if repeated != expected {
		t.Errorf("expected %q but got %q", expected, repeated)
	}
}
```

## 尝试运行测试

`./repeat_test.go:6:14: undefined: Repeat`

## 写最少量的代码让测试运行起来，并检查失败的测试输出

_保持纪律！_ 你现在不需要知道任何新东西就能让测试正确地失败。

你现在需要做的，只是写够能让它编译的代码，这样你才能检查你的测试是否写得好。

```go
package iteration

func Repeat(character string) string {
	return ""
}
```

知道你已经懂得足够多的 Go，可以为一些基础问题写测试，是不是挺好？这意味着你现在可以随心所欲地折腾生产代码，并知道它的行为是否符合你的期望。

`repeat_test.go:10: expected 'aaaaa' but got ''`

## 写足够的代码让它通过

`for` 的语法非常普通，与大多数类 C 语言一致。

```go
func Repeat(character string) string {
	var repeated string
	for i := 0; i < 5; i++ {
		repeated = repeated + character
	}
	return repeated
}
```

不像 C、Java、JavaScript 这些语言，for 语句的三个组成部分周围没有圆括号，而花括号 `{ }` 总是必需的。你可能会想这一行发生了什么

```go
	var repeated string
```

因为我们之前一直用 `:=` 来声明并初始化变量。然而，`:=` 只是[这两步的简写](https://gobyexample.com/variables)。这里我们只声明了一个 `string` 变量，所以是显式的写法。我们也可以用 `var` 来声明函数，后面会看到。

运行测试，它应该通过了。

更多 for 循环的变体在[这里](https://gobyexample.com/for)有描述。

## 重构

现在是重构的时候了，并引入另一个结构 `+=` 赋值运算符。

```go
const repeatCount = 5

func Repeat(character string) string {
	var repeated string
	for i := 0; i < repeatCount; i++ {
		repeated += character
	}
	return repeated
}
```

`+=` 称作 _"加且赋值运算符"_，把右操作数加到左操作数上，并把结果赋值给左操作数。它也可以用于其他类型，比如整数。

### 基准测试

在 Go 中编写[基准测试](https://golang.org/pkg/testing/#hdr-Benchmarks)是该语言的另一个一等公民特性，它和写测试非常相似。

```go
func BenchmarkRepeat(b *testing.B) {
	for b.Loop() {
		Repeat("a")
	}
}
```

你会看到这段代码和测试非常类似。

`testing.B` 让你可以使用 loop 函数。只要基准测试需要继续运行，`Loop()` 就会返回 true。

当基准测试代码被执行时，它会测量需要多长时间。在 `Loop()` 返回 false 之后，`b.N` 会包含运行的总迭代次数。

代码运行的次数对你来说应该不重要，框架会决定一个"好"的值，让你得到不错的结果。

要运行基准测试，执行 `go test -bench=.`（如果你在 Windows Powershell 里，就用 `go test -bench="."`）

```text
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           136 ns/op
PASS
```

`136 ns/op` 的意思是我们的函数平均运行时间是 136 纳秒（在我的电脑上）。挺不错的！为了测试这一点，它运行了 10000000 次。

**注意：** 默认情况下基准测试是顺序运行的。

只有循环体被计时；它会自动把基准测试计时排除掉准备和清理代码。一个典型的基准测试结构如下：

```go
func Benchmark(b *testing.B) {
	//... setup ...
	for b.Loop() {
		//... code to measure ...
	}
	//... cleanup ...
}
```

Go 中的字符串是不可变的，这意味着每次拼接（比如我们的 `Repeat` 函数中的拼接）都涉及复制内存来容纳新字符串。这会影响性能，特别是在大量字符串拼接时。

标准库提供了 `strings.Builder`[stringsBuilder] 类型，它最小化了内存复制。
它实现了一个 `WriteString` 方法，我们可以用它来拼接字符串：

```go
const repeatCount = 5

func Repeat(character string) string {
	var repeated strings.Builder
	for i := 0; i < repeatCount; i++ {
		repeated.WriteString(character)
	}
	return repeated.String()
}
```

**注意**：我们必须调用 `String` 方法来获取最终结果。

我们可以用 `BenchmarkRepeat` 来确认 `strings.Builder` 显著提升了性能。
运行 `go test -bench=. -benchmem`：

```text
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           25.70 ns/op           8 B/op           1 allocs/op
PASS
```

`-benchmem` 标志报告内存分配的相关信息：

* `B/op`：每次迭代分配的字节数
* `allocs/op`：每次迭代的内存分配次数

## 练习

* 修改测试，让调用者可以指定字符重复多少次，然后修复代码
* 写一个 `ExampleRepeat` 来给你的函数做文档
* 看看 [strings](https://golang.org/pkg/strings) 包。找一些你认为可能有用的函数，像我们这里一样写测试来体验它们。投入时间学习标准库，长期来看一定会有回报。

## 总结

* 更多 TDD 练习
* 学了 `for`
* 学了如何写基准测试

[stringsBuilder]: https://pkg.go.dev/strings#Builder
