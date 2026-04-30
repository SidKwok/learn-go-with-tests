# Hello, World

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/hello-world)**

按惯例，学习一门新语言的第一个程序是 [Hello, World](https://en.m.wikipedia.org/wiki/%22Hello,_World!%22_program)。

- 在你喜欢的位置创建一个文件夹
- 在里面新建一个名为 `hello.go` 的文件，并把下面的代码放进去

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, world")
}
```

要运行它，输入 `go run hello.go`。

## 它是怎么工作的

当你用 Go 写程序时，你会有一个名为 `main` 的包，里面定义了一个名为 `main` 的函数。包是把相关 Go 代码组织在一起的方式。

`func` 关键字定义了一个带名称和函数体的函数。

通过 `import "fmt"`，我们引入了一个包，它包含我们用来打印的 `Println` 函数。

## 怎么测试

你怎么测试这段代码？把你的"领域"代码与外部世界（副作用）分离是个好习惯。`fmt.Println` 是一个副作用（向标准输出打印），而我们传入的字符串是我们的领域。

那让我们把这两个关注点分离开，使代码更易测试

```go
package main

import "fmt"

func Hello() string {
	return "Hello, world"
}

func main() {
	fmt.Println(Hello())
}
```

我们用 `func` 创建了一个新函数，但这次我们在定义中加上了另一个关键字 `string`。这表示这个函数返回一个 `string`。

现在创建一个新文件 `hello_test.go`，我们将在其中为 `Hello` 函数编写测试

```go
package main

import "testing"

func TestHello(t *testing.T) {
	got := Hello()
	want := "Hello, world"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

## Go modules？

下一步是运行测试。在终端输入 `go test`。如果测试通过了，你大概用的是较早版本的 Go。然而如果你用的是 Go 1.16 或更高版本，测试很可能跑不起来。相反，你会在终端看到类似下面的错误信息：

```shell
$ go test
go: cannot find main module; see 'go help modules'
```

问题出在哪？一句话：[modules](https://blog.golang.org/go116-module-changes)。幸好，问题很容易修。在终端输入 `go mod init example.com/hello`。这会创建一个新文件，内容如下：

```
module example.com/hello

go 1.16
```

这个文件告诉 `go` 工具关于你代码的关键信息。如果你打算分发你的应用，你会在其中包含代码可下载的位置以及依赖信息。模块名 example\.com\/hello 通常指向模块可以被找到和下载的 URL。为了与我们即将使用的工具兼容，请确保你的模块名中带一个点，就像 example\.com/hello 中 .com 里的那个点。目前你的模块文件很简单，可以保持这样。要更深入了解 modules，[可以查看 Golang 文档中的参考](https://golang.org/doc/modules/gomod-ref)。我们现在可以回到测试和学习 Go 了，因为即使在 Go 1.16 上，测试也应该能跑起来了。

在后续章节中，你需要在每个新文件夹里运行 `go mod init SOMENAME`，然后才能运行 `go test` 或 `go build` 这类命令。

## 回到测试

在终端运行 `go test`。它应该通过了！为了确认，可以试着故意把 `want` 字符串改错来让测试失败。

注意你不需要在多个测试框架之间挑选，再去琢磨怎么安装它们。你需要的一切都内置在语言里，语法也和你写的其他代码一致。

### 编写测试

写测试就像写函数一样，只有几条规则

* 它需要在一个名字形如 `xxx_test.go` 的文件里
* 测试函数的名字必须以 `Test` 开头
* 测试函数只接受一个参数 `t *testing.T`
* 要使用 `*testing.T` 类型，你需要 `import "testing"`，就像我们在另一个文件里 import `fmt` 那样

目前你只需要知道，类型为 `*testing.T` 的 `t` 是你与测试框架交互的"钩子"，你可以通过它做一些事，比如想让测试失败时调用 `t.Fail()`。

我们已经涉及了一些新主题：

#### `if`
Go 中的 if 语句和其他编程语言非常相似。

#### 声明变量

我们用 `varName := value` 的语法声明一些变量，这样我们可以在测试里复用这些值，提升可读性。

#### `t.Errorf`

我们调用了 `t` 上的 `Errorf` _方法_，它会打印一条消息并使测试失败。`f` 代表 format（格式化），它允许我们用占位符 `%q` 把值插入字符串中构建消息。当你让测试失败时，应该能看清它是怎么工作的。

你可以在 [fmt 文档](https://pkg.go.dev/fmt#hdr-Printing) 里了解更多关于占位符的内容。在测试中，`%q` 非常有用，因为它会用双引号把你的值包起来。

我们之后会探讨方法和函数的区别。

### Go 的文档

Go 另一个让生活更便利的特性是它的文档。我们刚才在官方包查阅网站上看到了 fmt 包的文档，Go 也提供了快速离线获取文档的方式。

Go 自带一个工具 doc，它能让你查看任何安装在系统上的包，或者你正在开发的模块。要查看刚才那些 Printing 动词的文档：

```
$ go doc fmt
package fmt // import "fmt"

Package fmt implements formatted I/O with functions analogous to C's printf and
scanf. The format 'verbs' are derived from C's but are simpler.

# Printing

The verbs:

General:

    %v	the value in a default format
    	when printing structs, the plus flag (%+v) adds field names
    %#v	a Go-syntax representation of the value
    %T	a Go-syntax representation of the type of the value
    %%	a literal percent sign; consumes no value
...
```

Go 的第二个查看文档的工具是 pkgsite 命令，它驱动了 Go 的官方包查阅网站。你可以用 `go install golang.org/x/pkgsite/cmd/pkgsite@latest` 安装 pkgsite，然后用 `pkgsite -open .` 来运行。Go 的 install 命令会从对应仓库下载源代码并编译成可执行二进制。在 Go 的默认安装中，可执行文件会在 Linux 和 macOS 的 `$HOME/go/bin` 中，Windows 上则是 `%USERPROFILE%\go\bin`。如果你还没把这些路径加到 $PATH 里，建议加一下，会让运行通过 go install 安装的命令更方便。

绝大部分标准库都有非常优秀的文档和示例。访问 [http://localhost:8080/testing](http://localhost:8080/testing) 看看你能用到什么是值得的。


### Hello, YOU

现在我们有了测试，可以放心地迭代我们的软件了。

在上一个例子里，我们是 _在代码写完之后_ 才写测试的，目的是让你看到一个写测试和声明函数的例子。从这里开始，我们会 _先写测试_。

我们的下一个需求是允许指定问候的对象。

让我们从把这些需求转成测试开始。这是基本的测试驱动开发，能确保我们的测试 _确实_ 在测试我们想要的东西。当你事后才补写测试时，存在一个风险：即使代码没按预期工作，你的测试也可能继续通过。

```go
package main

import "testing"

func TestHello(t *testing.T) {
	got := Hello("Chris")
	want := "Hello, Chris"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

现在运行 `go test`，你应该会看到一个编译错误

```text
./hello_test.go:6:18: too many arguments in call to Hello
    have (string)
    want ()
```

使用像 Go 这样的静态类型语言时，_听编译器的话_ 很重要。编译器明白你的代码应当如何拼接和工作，所以你不必都自己想清楚。

在这个例子里，编译器在告诉你需要做什么才能继续。我们必须修改 `Hello` 函数让它接受一个参数。

修改 `Hello` 函数让它接受一个 string 类型的参数

```go
func Hello(name string) string {
	return "Hello, world"
}
```

如果你再次运行测试，你的 `hello.go` 会编译失败，因为你没有传入参数。传入 "world" 让它能编译。

```go
func main() {
	fmt.Println(Hello("world"))
}
```

现在运行测试，你应该会看到类似这样的输出

```text
hello_test.go:10: got 'Hello, world' want 'Hello, Chris''
```

我们终于有了一个能编译的程序，但根据测试它还没满足我们的需求。

让我们通过使用 name 参数并把它和 `Hello,` 拼接起来让测试通过

```go
func Hello(name string) string {
	return "Hello, " + name
}
```

当你再运行测试时，它们应该能通过了。通常作为 TDD 循环的一部分，我们现在应该 _重构_。

### 关于版本控制的提醒

到这里如果你在使用版本控制（你应该用！）的话，我会把代码现在的状态 `commit` 一下。我们有了能工作的软件并有测试支持。

不过我 _不会_ 推送到 main 分支，因为我打算接下来重构。在这个时间点提交一下挺好的，万一你重构搞乱了，你总能回到能工作的版本。

这里没什么可重构的，但我们可以引入另一个语言特性，_常量_。

### 常量

常量是这样定义的

```go
const englishHelloPrefix = "Hello, "
```

我们现在可以重构代码

```go
const englishHelloPrefix = "Hello, "

func Hello(name string) string {
	return englishHelloPrefix + name
}
```

重构后，重新运行测试以确保没有破坏任何东西。

值得思考的是，创建常量来表达值的含义有时也能帮助提升性能。

## Hello, world... 再来一次

下一个需求是当我们的函数被传入空字符串时，默认打印 "Hello, World"，而不是 "Hello, "。

先写一个失败的测试

```go
func TestHello(t *testing.T) {
	t.Run("saying hello to people", func(t *testing.T) {
		got := Hello("Chris")
		want := "Hello, Chris"

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})
	t.Run("say 'Hello, World' when an empty string is supplied", func(t *testing.T) {
		got := Hello("")
		want := "Hello, World"

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})
}
```

这里我们引入了测试武器库里的另一个工具：子测试。有时把围绕"某件事物"的测试归在一起，再用子测试描述不同场景，会很有用。

这种方式的一个好处是你可以设置一些可以被其他测试共享的代码。

测试还是失败的，让我们用 `if` 来修代码。

```go
const englishHelloPrefix = "Hello, "

func Hello(name string) string {
	if name == "" {
		name = "World"
	}
	return englishHelloPrefix + name
}
```

如果我们运行测试，应该会看到它满足了新需求，并且我们没有意外地破坏其他功能。

测试 _作为代码该做什么的清晰规范_ 是很重要的。但当我们检查消息是否符合预期时，存在重复代码。

重构 _不仅是_ 针对生产代码的！

既然测试通过了，我们可以也应该重构我们的测试。

```go
func TestHello(t *testing.T) {
	t.Run("saying hello to people", func(t *testing.T) {
		got := Hello("Chris")
		want := "Hello, Chris"
		assertCorrectMessage(t, got, want)
	})

	t.Run("empty string defaults to 'world'", func(t *testing.T) {
		got := Hello("")
		want := "Hello, World"
		assertCorrectMessage(t, got, want)
	})

}

func assertCorrectMessage(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

我们做了什么？

我们把断言重构成一个新函数。这减少了重复，并提升了测试的可读性。我们需要传入 `t *testing.T`，这样在需要时可以让测试代码失败。

对于辅助函数，最好接受 `testing.TB`，它是一个接口，`*testing.T` 和 `*testing.B` 都满足这个接口，因此你可以从测试或基准测试中调用辅助函数（如果"接口"这样的词现在对你来说一无所知，别担心，后面会讲到）。

`t.Helper()` 用来告诉测试套件这个方法是辅助函数。这样做之后，当它失败时，报告的行号会指向 _我们的函数调用_，而不是测试辅助函数内部。这能帮助其他开发者更容易追踪问题。如果你还不理解，把它注释掉，让一个测试失败，观察测试输出。Go 中的注释是给代码增加额外信息的好方法，或者像这里这样，是快速告诉编译器忽略某行的方式。你可以通过在行首加两个斜杠 `//` 来注释掉 `t.Helper()` 那行代码。你会看到那行变灰或变成与其他代码不同的颜色，表示它已被注释。

当你有多个相同类型的参数（这里是两个 string）时，相比 `(got string, want string)`，你可以缩写成 `(got, want string)`。

### 回到版本控制

现在我们对代码满意了，我会 amend（修改）之前那个提交，这样我们只签入了带测试的整洁版本代码。

### 纪律

让我们再过一遍这个循环

* 写一个测试
* 让编译器通过
* 运行测试，看到它失败，并检查错误信息是否有意义
* 写刚好够让测试通过的代码
* 重构

表面上看这可能很繁琐，但坚持这个反馈循环很重要。

它不仅能确保你有 _相关的测试_，还能帮你在测试的安全保障下通过重构来 _设计出好的软件_。

看到测试失败是一个重要的检查，因为它也让你看到错误信息长什么样。作为开发者，当失败的测试不能清晰地告诉你问题所在时，要在代码库里工作会非常困难。

通过确保测试 _快速_、并配置好工具让运行测试变得简单，你就能在写代码时进入心流状态。

不写测试的话，你就承诺要通过运行软件手动检查代码，这会打断你的心流。你不会节省任何时间，从长远来看尤其如此。

## 继续！更多需求

天哪，又来更多需求了。我们现在需要支持第二个参数，用来指定问候的语言。如果传入一个我们不认识的语言，就默认用英文。

我们应该有信心可以轻松用 TDD 把这个功能补齐！

写一个测试，让用户传入西班牙语。把它加到现有的测试套件里。

```go
	t.Run("in Spanish", func(t *testing.T) {
		got := Hello("Elodie", "Spanish")
		want := "Hola, Elodie"
		assertCorrectMessage(t, got, want)
	})
```

记得别作弊！_先写测试_。当你尝试运行测试时，编译器 _应该_ 会抱怨，因为你用两个参数调用了 `Hello`，而不是一个。

```text
./hello_test.go:27:19: too many arguments in call to Hello
    have (string, string)
    want (string)
```

通过给 `Hello` 加另一个 string 参数来修复编译问题

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}
	return englishHelloPrefix + name
}
```

当你再尝试运行测试时，它会抱怨在其他测试和 `hello.go` 中调用 `Hello` 时没有传够参数。

```text
./hello.go:15:19: not enough arguments in call to Hello
    have (string)
    want (string, string)
```

通过传入空字符串来修复它们。现在所有测试都应该能编译 _并_ 通过，除了我们的新场景

```text
hello_test.go:29: got 'Hello, Elodie' want 'Hola, Elodie'
```

这里我们可以用 `if` 检查 language 是否等于 "Spanish"，如果是，就改变消息

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	if language == "Spanish" {
		return "Hola, " + name
	}
	return englishHelloPrefix + name
}
```

测试现在应该能通过。

现在是 _重构_ 的时间。你应该看到代码里有些问题，"魔法"字符串，其中一些还重复了。你自己试着去重构它，每次改动都要重新运行测试以确保你的重构没有破坏任何东西。

```go
	const spanish = "Spanish"
	const englishHelloPrefix = "Hello, "
	const spanishHelloPrefix = "Hola, "

	func Hello(name string, language string) string {
		if name == "" {
			name = "World"
		}

		if language == spanish {
			return spanishHelloPrefix + name
		}
		return englishHelloPrefix + name
	}
```

### 法语

* 写一个测试，断言如果你传入 `"French"`，会得到 `"Bonjour, "`
* 看着它失败，检查错误信息易读
* 在代码里做最小的合理改动

你大概会写出下面这样的东西

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	if language == spanish {
		return spanishHelloPrefix + name
	}
	if language == french {
		return frenchHelloPrefix + name
	}
	return englishHelloPrefix + name
}
```

## `switch`

当你有很多 `if` 语句去检查同一个值时，常用的做法是改用 `switch`。我们可以用 `switch` 重构代码，让它更易读，并且如果以后想加更多语言支持时也更易扩展

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	prefix := englishHelloPrefix

	switch language {
	case spanish:
		prefix = spanishHelloPrefix
	case french:
		prefix = frenchHelloPrefix
	}

	return prefix + name
}
```

写一个测试加上你选择的语言的问候，你应该能看到扩展我们这个 _出色的_ 函数有多简单。

### 最后...再...重构一次？

你也可以认为我们的函数变得有点大了。最简单的重构是把一些功能抽成另一个函数。

```go

const (
	spanish = "Spanish"
	french  = "French"

	englishHelloPrefix = "Hello, "
	spanishHelloPrefix = "Hola, "
	frenchHelloPrefix  = "Bonjour, "
)

func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	return greetingPrefix(language) + name
}

func greetingPrefix(language string) (prefix string) {
	switch language {
	case french:
		prefix = frenchHelloPrefix
	case spanish:
		prefix = spanishHelloPrefix
	default:
		prefix = englishHelloPrefix
	}
	return
}
```

几个新概念：

* 在我们的函数签名里我们使用了 _命名返回值_ `(prefix string)`。
* 这会在你的函数里创建一个名为 `prefix` 的变量。
  * 它会被赋予"零值"。这取决于类型，例如 `int` 是 0，`string` 是 `""`。
    * 你可以通过仅调用 `return` 而不是 `return prefix` 来返回它当前的值。
  * 它会显示在你函数的 Go Doc 中，可以让你代码的意图更清晰。
* 当所有 `case` 语句都不匹配时，switch 会走 `default` 分支。
* 函数名以小写字母开头。在 Go 中，公开函数以大写字母开头，私有函数以小写字母开头。我们不希望算法的内部细节暴露给外部，所以把这个函数设为私有。
* 此外，我们可以把常量用一个块组织在一起，而不是各自单独声明。为了可读性，相关常量组之间用一空行分隔是个好主意。

## 总结

谁能想到从 `Hello, world` 中能挖出这么多东西？

到现在为止你应该理解了：

### Go 的部分语法

* 编写测试
* 声明带参数和返回类型的函数
* `if`、`const` 和 `switch`
* 声明变量和常量

### TDD 流程，以及 _为什么_ 这些步骤很重要

* _写一个失败的测试并看到它失败_，这样我们就知道我们写的是 _符合需求_ 的测试，并且能看到它产生 _易于理解的失败描述_
* 写最少量的代码让测试通过，这样我们就知道我们有了能工作的软件
* _然后_ 重构，在测试的安全保障下，确保我们的代码精雕细琢、易于维护

在我们这个例子里，我们以小而易懂的步骤，从 `Hello()` 走到了 `Hello("name")`，再到 `Hello("name", "French")`。

当然，这与"真实世界"的软件相比是微不足道的，但原则依然成立。TDD 是一项需要练习才能掌握的技能，但通过把问题拆解成可以测试的更小组件，你会更轻松地写出软件。
