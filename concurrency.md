# 并发

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/concurrency)**

情景如下：一位同事写了一个名叫 `CheckWebsites` 的函数，
它检查一组 URL 的状态。

```go
package concurrency

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		results[url] = wc(url)
	}

	return results
}
```

它返回一个 map，把每个被检查的 URL 映射到一个布尔值：`true` 表示响应良好；
`false` 表示响应不好。

你还得传入一个 `WebsiteChecker`，它接收单个 URL 并返回
一个布尔值。函数用它来检查所有网站。

使用 [依赖注入][DI] 让他们能够在不发起真实 HTTP 调用的情况下
测试这个函数，使得测试可靠且快速。

下面是他们写的测试：

```go
package concurrency

import (
	"reflect"
	"testing"
)

func mockWebsiteChecker(url string) bool {
	return url != "waat://furhurterwe.geds"
}

func TestCheckWebsites(t *testing.T) {
	websites := []string{
		"http://google.com",
		"http://blog.gypsydave5.com",
		"waat://furhurterwe.geds",
	}

	want := map[string]bool{
		"http://google.com":          true,
		"http://blog.gypsydave5.com": true,
		"waat://furhurterwe.geds":    false,
	}

	got := CheckWebsites(mockWebsiteChecker, websites)

	if !reflect.DeepEqual(want, got) {
		t.Fatalf("wanted %v, got %v", want, got)
	}
}
```

这个函数已经投入生产，被用来检查数百个网站。但
你的同事开始收到投诉说它太慢，所以他们请你
帮忙提速。

## 写一个测试

让我们用一个基准测试来测 `CheckWebsites` 的速度，这样我们就能看到
我们改动的效果。

```go
package concurrency

import (
	"testing"
	"time"
)

func slowStubWebsiteChecker(_ string) bool {
	time.Sleep(20 * time.Millisecond)
	return true
}

func BenchmarkCheckWebsites(b *testing.B) {
	urls := make([]string, 100)
	for i := 0; i < len(urls); i++ {
		urls[i] = "a url"
	}

	for b.Loop() {
		CheckWebsites(slowStubWebsiteChecker, urls)
	}
}
```

这个基准测试用 100 个 URL 的切片测试 `CheckWebsites`，并使用
一个新的 `WebsiteChecker` 假实现。`slowStubWebsiteChecker` 是
故意慢的。它使用 `time.Sleep` 等待整整 20 毫秒，
然后返回 true。


当我们用 `go test -bench=.` 运行基准测试时（如果你用的是 Windows Powershell，则用 `go test -bench="."`）：

```sh
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v0
BenchmarkCheckWebsites-4               1        2249228637 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v0        2.268s
```

`CheckWebsites` 的基准测试结果是 2249228637 纳秒——大约两秒
四分之一。

我们来试着让它更快。

### 写足够的代码让测试通过

现在我们终于可以谈谈并发了，就下面要讲的而言，并发的意思是
"同时进行多件事"。这是我们每天都在自然而然做的事。

例如，今天早上我泡了一杯茶。我把水壶放到炉子上烧水，
在等水开的时候，我从冰箱里拿出牛奶，从橱柜里拿出茶叶，
找到我最喜欢的杯子，把茶包放进杯子里，
然后等水开了之后，把水倒进杯子。

我 _没做_ 的是把水壶放上去后呆呆地盯着水壶，
直到它烧开，再做其他所有事。

如果你能理解为什么第一种方式泡茶更快，那么你就能
理解我们要怎么让 `CheckWebsites` 更快。我们不会等
一个网站响应再向下一个网站发请求，而是告诉
我们的电脑在等待时就发出下一个请求。

通常在 Go 中，当我们调用一个函数 `doSomething()` 时，我们会等它返回
（即使它没有值要返回，我们仍然等它完成）。我们说
这种操作是 *阻塞* 的——它让我们等它完成。在 Go 中
不阻塞的操作会运行在一个独立的 *进程* 中，称为 *goroutine*。
把一个进程想象成从上到下读 Go 代码的页面，被调用时
"进入"每个函数读它的内容。当一个独立的进程开始时，
就像另一个读者开始在函数内部阅读，
原来的读者继续往下读页面。

要告诉 Go 启动一个新的 goroutine，我们把函数调用变成 `go`
语句，方法是把关键字 `go` 放在它前面：`go doSomething()`。

```go
package concurrency

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func() {
			results[url] = wc(url)
		}()
	}

	return results
}
```

因为启动 goroutine 的唯一方式是把 `go` 放在函数调用前面，
所以当我们想启动一个 goroutine 时，我们经常使用 *匿名函数*。
匿名函数字面量看起来和普通函数声明一模一样，
但没有名字（不出所料）。你可以在上面 `for` 循环的循环体里看到一个。

匿名函数有一些让它有用的特性，其中两个我们在上面用到了。
首先，它们可以在被声明时同时被执行——这是匿名函数末尾的 `()` 在做的事。
其次，它们保持对其定义所在词法作用域的访问——
你声明匿名函数时所有可用的变量在
函数体内也都可用。

上面匿名函数的函数体和原来循环体一模一样。
唯一的区别是循环的每次迭代都会启动一个新的
goroutine，与当前进程（`WebsiteChecker` 函数）并发执行。
每个 goroutine 都会把自己的结果加到 results map 里。

但当我们运行 `go test`：

```sh
--- FAIL: TestCheckWebsites (0.00s)
        CheckWebsites_test.go:31: Wanted map[http://google.com:true http://blog.gypsydave5.com:true waat://furhurterwe.geds:false], got map[]
FAIL
exit status 1
FAIL    github.com/gypsydave5/learn-go-with-tests/concurrency/v1        0.010s

```

### 暂时插播一下并发的世界……

你可能不会得到这样的结果。你可能会得到一条 panic 信息，
我们待会会讲到。如果你得到了那个，别担心，继续
运行测试直到你 _确实_ 得到上面的结果。或者就当你得到了。
随你。欢迎来到并发的世界：处理不当的话很难
预测会发生什么。别担心——这就是我们写测试的原因，
帮我们知道什么时候我们对并发处理是可预测的。

### ……我们回来了。

我们被原来的测试 `CheckWebsites` 抓住了，它现在返回了
一个空 map。哪里出错了？

我们 `for` 循环启动的 goroutine 都没有足够的时间把
它们的结果加入 `results` map；`CheckWebsites` 函数对它们来说太快了，
它返回了仍然为空的 map。

要解决这个问题，我们可以等所有 goroutine 完成它们的工作，再
返回。两秒应该够了，对吧？

```go
package concurrency

import "time"

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func() {
			results[url] = wc(url)
		}()
	}

	time.Sleep(2 * time.Second)

	return results
}
```

如果你运气好，你会得到：

```sh
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v1        2.012s
```

但如果你运气不好（如果你跟基准测试一起运行更可能这样，因为你会有更多次尝试）

```sh
fatal error: concurrent map writes

goroutine 8 [running]:
runtime.throw(0x12c5895, 0x15)
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/panic.go:605 +0x95 fp=0xc420037700 sp=0xc4200376e0 pc=0x102d395
runtime.mapassign_faststr(0x1271d80, 0xc42007acf0, 0x12c6634, 0x17, 0x0)
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:783 +0x4f5 fp=0xc420037780 sp=0xc420037700 pc=0x100eb65
github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1(0xc42007acf0, 0x12d3938, 0x12c6634, 0x17)
        /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x71 fp=0xc4200377c0 sp=0xc420037780 pc=0x12308f1
runtime.goexit()
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/asm_amd64.s:2337 +0x1 fp=0xc4200377c8 sp=0xc4200377c0 pc=0x105cf01
created by github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker
        /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xa1

        ... many more scary lines of text ...
```

这又长又吓人，但我们要做的就是深吸一口气，读读
这个堆栈跟踪：`fatal error: concurrent map writes`。有时候，当我们运行
测试时，两个 goroutine 在完全相同的时刻向 results map 写入。
Go 中的 map 不喜欢有多个东西同时往里写，
所以 `fatal error`。

这是一个 _数据竞争_，当两个或多个 goroutine 并发地访问同一个内存位置，并且其中至少一次访问是写操作时，就会发生这个 bug。因为我们无法精确控制每个 goroutine 何时执行，所以我们有可能让多个 goroutine 在完全相同的时刻往 `results` map 里写。Go 的 map 不支持并发写入，所以运行时会抛出一个致命错误以防止内存损坏。

Go 可以通过其内置的 [_竞态检测器_][godoc_race_detector] 帮我们发现竞态条件。
要启用这个特性，运行测试时加上 `race` 标志：`go test -race`。

你应该会得到类似下面的输出：

```sh
==================
WARNING: DATA RACE
Write at 0x00c420084d20 by goroutine 8:
  runtime.mapassign_faststr()
      /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:774 +0x0
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x82

Previous write at 0x00c420084d20 by goroutine 7:
  runtime.mapassign_faststr()
      /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:774 +0x0
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x82

Goroutine 8 (running) created at:
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xc4
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.TestWebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker_test.go:27 +0xad
  testing.tRunner()
      /usr/local/Cellar/go/1.9.3/libexec/src/testing/testing.go:746 +0x16c

Goroutine 7 (finished) created at:
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xc4
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.TestWebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker_test.go:27 +0xad
  testing.tRunner()
      /usr/local/Cellar/go/1.9.3/libexec/src/testing/testing.go:746 +0x16c
==================
```

细节再次很难读——但 `WARNING: DATA RACE` 相当
明确无误。深入错误信息我们能看到两个不同的
goroutine 在向一个 map 写入：

`Write at 0x00c420084d20 by goroutine 8:`

正在向同一块内存写入，与

`Previous write at 0x00c420084d20 by goroutine 7:`

是相同的位置。

除此之外，我们还能看到写发生在哪一行代码：

`/Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12`

以及 goroutine 7 和 8 启动的那行代码：

`/Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11`

你需要知道的一切都打印到了你的终端——你只需要
有耐心读它就行了。

### Channel

我们可以通过用 _channel_ 协调 goroutine 来解决这个数据竞争。
channel 是 Go 的一种数据结构，可以接收和发送值。这些
操作以及它们的细节，让不同进程之间能够通信。

在这个场景里我们想思考的是父进程和它创建的、
用来运行 `WebsiteChecker` 函数的每个 goroutine 之间的通信。

```go
package concurrency

type WebsiteChecker func(string) bool
type result struct {
	string
	bool
}

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)
	resultChannel := make(chan result)

	for _, url := range urls {
		go func() {
			resultChannel <- result{url, wc(url)}
		}()
	}

	for i := 0; i < len(urls); i++ {
		r := <-resultChannel
		results[r.string] = r.bool
	}

	return results
}
```

除了 `results` map，我们现在还有一个 `resultChannel`，
我们用同样的方式 `make` 它。`chan result` 是 channel 的类型——一个 `result` 的 channel。
新类型 `result` 用来把 `WebsiteChecker` 的返回值
和被检查的 url 关联起来——它是一个由 `string` 和
`bool` 组成的结构体。因为我们不需要给任一值命名，
所以它们在结构体内都是匿名的；当难以为一个值命名时这会很有用。

现在当我们迭代 url 时，不再直接写入 `map`，
而是用 _发送语句_ 把每次调用 `wc` 的 `result` 结构体
发送到 `resultChannel`。它使用 `<-` 操作符，
左边是 channel，右边是值：

```go
// Send statement
resultChannel <- result{url, wc(url)}
```

下一个 `for` 循环对每个 url 各迭代一次。在内部我们使用
一个 _接收表达式_，它把从 channel 接收到的值赋给
一个变量。它也使用 `<-` 操作符，但现在两个操作数
反过来了：channel 在右边，我们要赋值的变量
在左边：

```go
// Receive expression
r := <-resultChannel
```

然后我们用收到的 `result` 来更新 map。

通过把结果发送到一个 channel，我们可以控制每次写入
results map 的时机，确保它是一次发生一个。虽然每次
对 `wc` 的调用，以及每次向 result channel 的发送，都在它自己的进程里
并发发生，但当我们用接收表达式从 result channel
取出值时，每个结果都是一次处理一个。

我们对想要加速的代码部分使用了并发，同时
确保不能同时发生的部分仍然按线性方式发生。
而我们通过使用 channel 在涉及的多个进程之间进行通信。

当我们运行基准测试：

```sh
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v2
BenchmarkCheckWebsites-8             100          23406615 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v2        2.377s
```
23406615 纳秒——0.023 秒，大约是
原始函数的一百倍快。大获成功。

## 总结

这次练习在 TDD 上比平时轻一些。某种程度上我们
一直在对 `CheckWebsites` 函数做一次长长的重构；
输入和输出从未改变，它只是变快了。但我们已有的测试，
以及我们写的基准测试，让我们在重构 `CheckWebsites` 时
保持对软件仍能工作的信心，同时
表明它确实变快了。

在让它变快的过程中，我们学到了

- *goroutine*，Go 中并发的基本单位，让我们能管理多个
  网站检查请求。
- *匿名函数*，我们用它来启动每个并发进程
  来检查网站。
- *channel*，帮助组织和控制不同进程之间的通信，
  让我们避免了 *竞态条件* 的 bug。
- *竞态检测器*，帮我们调试并发代码中的问题

### 让它变快

一种敏捷构建软件的表述方式（经常被错误地归到 Kent Beck 名下）是：

> [先让它工作，再让它正确，再让它快][wrf]

其中"工作"是让测试通过，"正确"是重构代码，
"快"是优化代码使其例如运行得更快。我们只能
在让它工作并让它正确之后才能"让它快"。我们很幸运，给我们的代码已经被证明能工作，
也不需要重构。我们绝不应该在前两步完成之前
就尝试"让它快"，因为

> [过早优化是万恶之源][popt]
> -- Donald Knuth

[DI]: dependency-injection.md
[wrf]: http://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast
[godoc_race_detector]: https://blog.golang.org/race-detector
[popt]: http://wiki.c2.com/?PrematureOptimization
