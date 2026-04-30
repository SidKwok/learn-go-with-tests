# Dependency Injection

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/di)**

我们假定你已经阅读过[结构体那一节](./structs-methods-and-interfaces.md)，因为本章会需要一些关于接口的理解。

编程社区里关于依赖注入有 _很多_ 误解。希望本指南能向你展示

* 你不需要框架
* 它不会让你的设计过度复杂化
* 它有助于测试
* 它能帮你写出优秀的、通用目的的函数。

我们想写一个跟某人打招呼的函数，就像我们在 hello-world 那一章做的一样，但这次我们要测试 _实际的打印_ 行为。

简单回顾一下，那个函数大概长这样

```go
func Greet(name string) {
	fmt.Printf("Hello, %s", name)
}
```

但我们要怎么测试呢？调用 `fmt.Printf` 会打印到 stdout，用测试框架很难捕获。

我们要做的就是能 **注入**（不过是个时髦词，意思就是"传入"）打印这个依赖。

**我们的函数不需要关心打印 _发生在哪里_ 或 _怎么发生_，所以应该接受一个 _接口_ 而不是具体的类型。**

这样做之后，我们可以把实现换成打印到我们能控制的地方，进而测试它。"现实生活"里你会注入一个写到 stdout 的实现。

如果你看 [`fmt.Printf`](https://pkg.go.dev/fmt#Printf) 的源代码，就能找到我们的切入点

```go
// It returns the number of bytes written and any write error encountered.
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}
```

有意思！在内部，`Printf` 只是调用了 `Fprintf` 并传入 `os.Stdout`。

`os.Stdout` _到底_ 是什么？`Fprintf` 期望第 1 个参数传入什么？

```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

一个 `io.Writer`

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

由此我们可以推断 `os.Stdout` 实现了 `io.Writer`；`Printf` 把 `os.Stdout` 传给 `Fprintf`，而 `Fprintf` 期望一个 `io.Writer`。

随着你写更多 Go 代码，你会发现这个接口经常出现，因为它是表达"把这些数据放到某处"非常通用的接口。

所以我们知道在底层我们最终是用 `Writer` 把问候发送到某处的。让我们利用这个已有的抽象，让代码可测试且更具复用性。

## 先写测试

```go
func TestGreet(t *testing.T) {
	buffer := bytes.Buffer{}
	Greet(&buffer, "Chris")

	got := buffer.String()
	want := "Hello, Chris"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

`bytes` 包里的 `Buffer` 类型实现了 `Writer` 接口，因为它有 `Write(p []byte) (n int, err error)` 方法。

所以我们在测试里用它作为 `Writer` 传入，调用 `Greet` 之后再检查写进去的内容

## 试着运行测试

测试无法编译

```text
./di_test.go:10:2: undefined: Greet
```

## 写最少量的代码让测试运行起来，并检查失败的输出

_听编译器的话_，把问题修了。

```go
func Greet(writer *bytes.Buffer, name string) {
	fmt.Printf("Hello, %s", name)
}
```

`Hello, Chris di_test.go:16: got '' want 'Hello, Chris'`

测试失败了。注意名字被打印出来了，但去到了 stdout。

## 写足够的代码让测试通过

用 writer 把问候发送到测试里的 buffer。记住 `fmt.Fprintf` 跟 `fmt.Printf` 类似，但它接受一个 `Writer` 来发送字符串，而 `fmt.Printf` 默认发送到 stdout。

```go
func Greet(writer *bytes.Buffer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}
```

测试现在通过了。

## 重构

之前编译器告诉我们要传入一个指向 `bytes.Buffer` 的指针。这在技术上是对的，但不是很有用。

为了证明这一点，试着把 `Greet` 函数接到一个 Go 应用里，让它打印到 stdout。

```go
func main() {
	Greet(os.Stdout, "Elodie")
}
```

`./di.go:14:7: cannot use os.Stdout (type *os.File) as type *bytes.Buffer in argument to Greet`

正如前面讨论过的，`fmt.Fprintf` 允许你传入一个 `io.Writer`，我们知道 `os.Stdout` 和 `bytes.Buffer` 都实现了它。

如果我们把代码改成使用更通用的接口，那它在测试和应用程序里就都能用了。

```go
package main

import (
	"fmt"
	"io"
	"os"
)

func Greet(writer io.Writer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}

func main() {
	Greet(os.Stdout, "Elodie")
}
```

## 关于 io.Writer 的更多内容

我们还能用 `io.Writer` 把数据写到哪些地方？我们的 `Greet` 函数到底有多通用？

### 互联网

运行下面的代码

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
)

func Greet(writer io.Writer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}

func MyGreeterHandler(w http.ResponseWriter, r *http.Request) {
	Greet(w, "world")
}

func main() {
	log.Fatal(http.ListenAndServe(":5001", http.HandlerFunc(MyGreeterHandler)))
}
```

运行程序并访问 [http://localhost:5001](http://localhost:5001)。你会看到我们的 greeting 函数被使用了。

后面的章节会讲 HTTP 服务器，所以这里的细节不必太担心。

当你写一个 HTTP handler 时，你会拿到一个 `http.ResponseWriter` 和发起请求的 `http.Request`。当你实现服务器时，你用 writer 来 _写_ 响应。

你大概能猜到，`http.ResponseWriter` 也实现了 `io.Writer`，所以我们能在 handler 里复用我们的 `Greet` 函数。

## 总结

我们第一版代码不易测试，因为它把数据写到了我们无法控制的地方。

_在测试驱动下_ 我们重构了代码，通过 **注入依赖** 控制了数据 _写到哪里_，这让我们能够：

* **测试我们的代码** 如果一个函数难以 _轻松_ 测试，通常是因为依赖被硬连接进了函数 _或者_ 全局状态。比如，如果你有一个全局的数据库连接池被某种 service 层使用，多半很难测试，跑起来也慢。DI 会促使你（通过接口）注入一个数据库依赖，这样你就能在测试里用一个你能控制的东西把它 mock 掉。
* **分离关注点**，把 _数据去往何处_ 与 _怎么生成数据_ 解耦。如果你觉得某个方法/函数职责太多（既生成数据 _又_ 写数据库？又处理 HTTP 请求 _又_ 做领域级逻辑？），DI 多半就是你需要的工具。
* **让代码可在不同上下文中复用** 我们代码可以使用的第一个"新"上下文就是测试。再进一步，如果有人想用你的函数尝试别的，他们可以注入自己的依赖。

### mock 怎么办？听说 DI 需要它，而且它是邪恶的

mock 后面会详细介绍（它并不邪恶）。你用 mock 把注入的真实东西换成一个假的版本，便于在测试中控制和检查。不过在我们这个例子里，标准库已经有现成的东西可用。

### Go 标准库非常好，值得花时间研究

正因为对 `io.Writer` 接口有些熟悉，我们才能在测试中把 `bytes.Buffer` 用作我们的 `Writer`，而且我们还可以使用标准库里其他的 `Writer` 来在命令行应用或 web 服务器里使用我们的函数。

你越熟悉标准库，就越能看到这些通用接口，并能在自己的代码里复用它们，让你的软件能在多种场景下被复用。

这个例子深受 [The Go Programming language](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440) 一书中某一章的影响，如果你喜欢这一章，去把书买了吧！
