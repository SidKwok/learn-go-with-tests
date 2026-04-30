# 整数

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/integers)**

整数的工作方式正如你期望的那样。我们来写一个 `Add` 函数试试看。创建一个名为 `adder_test.go` 的测试文件，然后写下这段代码。

**注意：** 一个目录下的 Go 源文件只能有一个 `package`。请确保你的文件被组织在各自的包里。[这里有一个很好的解释。](https://dave.cheney.net/2014/12/01/five-suggestions-for-setting-up-a-go-project)

你的项目目录可能看起来像这样：

```
learnGoWithTests
    |
    |-> helloworld
    |    |- hello.go
    |    |- hello_test.go
    |
    |-> integers
    |    |- adder_test.go
    |
    |- go.mod
    |- README.md
```

## 先写测试

```go
package integers

import "testing"

func TestAdder(t *testing.T) {
	sum := Add(2, 2)
	expected := 4

	if sum != expected {
		t.Errorf("expected '%d' but got '%d'", expected, sum)
	}
}
```

你会注意到我们在格式化字符串里用的是 `%d` 而不是 `%q`。这是因为我们想打印的是一个整数，而不是字符串。

另外要注意，我们不再使用 main 包，而是定义了一个名为 `integers` 的包。顾名思义，这个包会把处理整数相关的函数（比如 `Add`）组织在一起。

## 尝试运行测试

运行测试 `go test`

查看编译错误

`./adder_test.go:6:9: undefined: Add`

## 写最少量的代码让测试运行起来，并检查失败的测试输出

写出刚好够让编译器通过的代码 _就够了_——记住，我们想确认测试是因为正确的原因失败的。

```go
package integers

func Add(x, y int) int {
	return 0
}
```

记住，当你有多个相同类型的参数（在我们这里是两个整数），相比 `(x int, y int)` 你可以缩写成 `(x, y int)`。

现在运行测试，我们应该满意于测试正确地报告了哪里出了问题。

`adder_test.go:10: expected '4' but got '0'`

如果你注意到了，我们在[上一章](hello-world.md#onelastrefactor)学过 _命名返回值_，但这里没用。一般来说，当从上下文看不清返回值含义时才应该使用它。在我们这里，`Add` 函数会把参数加起来已经相当清楚了。你可以参考[这篇 wiki](https://go.dev/wiki/CodeReviewComments#named-result-parameters) 了解更多细节。

## 写足够的代码让它通过

按照 TDD 最严格的标准，我们现在应该写 _刚好让测试通过的最少量代码_。一个咬文嚼字的程序员可能会这么干：

```go
func Add(x, y int) int {
	return 4
}
```

啊哈！又被坑了，TDD 是个骗局对吧？

我们可以再写一个测试，用一些不同的数字来强制让那个测试失败，但这感觉就像[一场猫鼠游戏](https://en.m.wikipedia.org/wiki/Cat_and_mouse)。

等我们对 Go 的语法更熟悉之后，我会介绍一种叫做 _"基于属性的测试"_ 的技术，它会让那些烦人的开发者闭嘴，并帮你找到 bug。

现在，让我们老老实实地修好它

```go
func Add(x, y int) int {
	return x + y
}
```

如果你重新运行测试，它们应该能通过了。

## 重构

_实际_ 代码里没有什么我们能真正改进的地方。

我们之前探讨过，给返回参数命名后，它会出现在文档里，也会出现在大多数开发者的文本编辑器里。

这很棒，因为它有助于你写的代码的可用性。一个用户能仅通过查看类型签名和文档就理解你代码的用法，是更可取的。

你可以通过注释给函数添加文档，这些注释会出现在 Go Doc 中，就像你查看标准库文档时看到的那样。

```go
// Add takes two integers and returns the sum of them.
func Add(x, y int) int {
	return x + y
}
```

### 可测试的示例

如果你真的想再多走一步，你可以写[可测试的示例](https://blog.golang.org/examples)。你会在标准库文档里看到很多这样的示例。

代码库之外的代码示例（比如 readme 文件里的）经常会因为没有被检查，而变得过时、与实际代码不符。

示例函数会在每次运行测试时被编译。因为这样的示例由 Go 编译器验证，你可以确信你文档里的示例总是能反映当前的代码行为。

示例函数以 `Example` 开头（很像测试函数以 `Test` 开头），并放在包的 `_test.go` 文件中。把下面这个 `ExampleAdd` 函数加到 `adder_test.go` 文件里。

```go
func ExampleAdd() {
	sum := Add(1, 5)
	fmt.Println(sum)
	// Output: 6
}
```

（如果你的编辑器不会自动帮你导入包，编译步骤会失败，因为你的 `adder_test.go` 里缺少 `import "fmt"`。强烈建议你研究一下，怎么让你正在使用的编辑器自动修复这类错误。）

加上这段代码会让示例出现在你的文档里，让你的代码更加易用。如果你的代码哪天发生了变更，导致示例不再有效，你的构建就会失败。

运行包的测试套件，我们能看到示例函数 `ExampleAdd` 在不需要我们做任何额外安排的情况下被执行了：

```bash
$ go test -v
=== RUN   TestAdder
--- PASS: TestAdder (0.00s)
=== RUN   ExampleAdd
--- PASS: ExampleAdd (0.00s)
```

注意注释的特殊格式 `// Output: 6`。虽然示例总会被编译，加上这条注释意味着示例还会被 _执行_。你不妨临时把 `// Output: 6` 这条注释删掉，然后运行 `go test`，你会看到 `ExampleAdd` 不再被执行。

没有输出注释的示例，可以用来演示那些不能作为单元测试运行的代码（比如访问网络的代码），同时保证示例至少能编译通过。

要查看示例文档，我们快速看一下 `pkgsite`。在导航到你项目的目录之前，先确保你已经通过运行下面这条命令安装了 `pkgsite`：`go install golang.org/x/pkgsite/cmd/pkgsite@latest`，然后运行 `pkgsite -open .`，它应该会为你打开一个浏览器窗口，地址指向 `http://localhost:8080`。在这里你会看到所有 Go 标准库包的列表，外加你已经安装的第三方包，在其中你应该能看到 `github.com/quii/learn-go-with-tests` 的示例文档。点开那个链接，然后看一下 `Integers`，再看 `func Add`，然后展开 `Example`，你应该能看到你为 `sum := Add(1, 5)` 所添加的示例。

如果你把带示例的代码发布到了一个公开的 URL，你可以在 [pkg.go.dev](https://pkg.go.dev/) 分享你代码的文档。例如，[这里](https://pkg.go.dev/github.com/quii/learn-go-with-tests/integers/v2) 是本章最终的 API。这个网页界面让你可以搜索标准库包和第三方包的文档。

## 总结

我们涵盖了：

*   更多 TDD 流程的练习
*   整数、加法
*   写更好的文档，让我们代码的使用者能快速理解其用法
*   如何使用我们代码的示例，作为测试的一部分被检查
