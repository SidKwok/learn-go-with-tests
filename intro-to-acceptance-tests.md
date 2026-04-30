# 验收测试入门

在 `$WORK`，我们一直在遇到需要为服务实现 "graceful shutdown"（优雅关闭）的需求。优雅关闭确保你的系统在被终止前能恰当地完成自己的工作。一个现实世界的类比是：有人尝试妥善结束一通电话再去开下一个会，而不是说到一半就直接挂断。

本章会在 HTTP 服务器的语境下介绍优雅关闭，以及如何编写"验收测试"来增强你对代码行为的信心。

读完之后，你会知道如何分享带有出色测试的包、降低维护成本、提升你工作质量的可信度。

## 关于 Kubernetes 的最少必要信息

我们的软件运行在 [Kubernetes](https://kubernetes.io/)（K8s）上。K8s 会因为各种原因终止 "pod"（实际上就是我们的软件），其中一个常见的原因是当我们推送了想要部署的新代码时。

我们对 [DORA 指标](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance) 设定了高标准，所以我们采用每天多次将小而渐进的改进和新功能部署到生产环境的方式工作。

当 K8s 想要终止一个 pod 时，它会启动一个["终止生命周期"](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace)，其中一部分就是给我们的软件发送一个 SIGTERM 信号。这是 K8s 在告诉我们的代码：

> 你需要把自己关掉，完成手头任何正在做的工作，因为在某个"宽限期"之后，我会发送 `SIGKILL`，到那时一切就结束了。

收到 `SIGKILL` 时，你的程序可能正在做的任何工作都会立即停止。

## 如果你没有"优雅"

视你软件的性质而定，如果你忽略 `SIGTERM`，可能会遇到问题。

我们具体的问题出现在传输中的 HTTP 请求上。当一个自动化测试在调用我们的 API 时，如果 K8s 决定停掉 pod，服务器就会死掉，测试拿不到服务器的响应，测试就会失败。

这会触发我们事故频道里的告警，需要某个开发者停下手中的事去处理这个问题。这种间歇性失败对我们团队来说是一种烦人的干扰。

这些问题并不是我们测试独有的。如果一个用户向你的系统发了请求，而进程在中途被终止，他们大概率会看到一个 5xx 的 HTTP 错误，这可不是你想给用户带来的体验。

## 当你拥有"优雅"

我们想做的是监听 `SIGTERM`，并且不要立刻杀掉服务器，而是：

- 停止接收任何新请求
- 让任何传输中的请求完成
- *然后*终止进程

## 怎么实现"优雅"

幸运的是，Go 已经有一个让服务器优雅关闭的机制：[net/http/Server.Shutdown](https://pkg.go.dev/net/http#Server.Shutdown)。

> Shutdown gracefully shuts down the server without interrupting any active connections. Shutdown works by first closing all open listeners, then closing all idle connections, and then waiting indefinitely for connections to return to idle and then shut down. If the provided context expires before the shutdown is complete, Shutdown returns the context's error, otherwise it returns any error returned from closing the Server's underlying Listener(s).

要处理 `SIGTERM`，我们可以使用 [os/signal.Notify](https://pkg.go.dev/os/signal#Notify)，它会把任何到来的信号发送到我们提供的 channel。

通过组合标准库这两个特性，你就可以监听 `SIGTERM` 并优雅关闭。

## 优雅关闭的包

为此，我写了 [https://pkg.go.dev/github.com/quii/go-graceful-shutdown](https://pkg.go.dev/github.com/quii/go-graceful-shutdown)。它提供了一个针对 `*http.Server` 的装饰器函数，会在检测到 `SIGTERM` 信号时调用其 `Shutdown` 方法

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// 这种情况通常发生在 ctx 截止前响应没写完，没什么办法
		log.Fatalf("uh oh, didn't shutdown gracefully, some responses may have been lost %v", err)
	}

	// 但愿你看到的总是这一条
	log.Println("shutdown gracefully! all responses were sent")
}
```

代码的具体细节对本次阅读不太重要，但在继续之前快速看一眼代码是值得的。

## 测试与反馈循环

当我们写 `gracefulshutdown` 包时，我们有单元测试来证明它行为正确，这给了我们大胆重构的信心。然而我们仍然没有"信心"它**真的**能工作。

我们加了一个 `cmd` 包，做了一个真实的程序来使用我们正在编写的包。我们会手动启动它、向它发起一个 HTTP 请求，然后给它发送一个 `SIGTERM` 看看会发生什么。

**作为工程师，你应该对手动测试感到不舒服**。
它无聊、无法扩展、不准确，而且浪费时间。如果你在写一个打算分享出去的包，但又想保持改动起来简单且低成本，手动测试是行不通的。

## 验收测试

如果你读过这本书的其他章节，那你写的多半都是"单元测试"。单元测试是一个绝佳的工具，可以让你无所畏惧地重构、推动良好的模块化设计、防止回归并提供快速反馈。

由于其本质，它们只测试系统的一小部分。通常，仅靠单元
测试本身*不足以*构成一个有效的测试策略。记住，我们希望我们的系统**始终可发布**。我们不能依赖手动测试，所以我们需要另一种测试：**验收测试**。

### 它们是什么？

验收测试是一种"黑盒测试"。它们有时也被称为
"功能测试"。它们应当像系统的用户一样去使用系统。

"黑盒"这个词指的是测试代码无法访问系统的内部实现，它只能使用其公开接口，并对它观察到的行为做断言。这意味着它们只能把系统作为一个整体来测试。

这是一个有利的特性，因为它意味着测试以与真实用户相同的方式运行系统，它不能使用任何特殊的"绕过手段"——那种手段虽然能让测试通过，却并没有真正证明你需要证明的东西。这与一种原则类似：偏向把单元测试文件放在独立的测试包里，例如使用 `package mypkg_test` 而不是 `package mypkg`。

### 验收测试的好处

- 当它们通过时，你就知道你的整个系统行为符合预期。
- 它们比手动测试更准确、更快、需要的精力更少。
- 写得好的话，它们就是关于你系统的、准确且经过验证的文档。它不会陷入"文档与系统真实行为分歧"的陷阱。
- 没有 mock！全是真实的。

### 相比单元测试可能的缺点

- 写起来成本高。
- 运行起来更慢。
- 它们依赖于系统的设计。
- 失败时，它们通常不会给你根因，调试可能会很困难。
- 它们不会反馈系统内部的质量。你
  写一坨垃圾代码也照样能让验收测试通过。
- 由于黑盒的本质，并非所有场景都适合演练。

因此，仅依赖验收测试是愚蠢的。它们不具备单元测试的许多优秀品质，并且一个有大量验收测试的系统在维护成本和交付前置时间方面通常会受影响。

#### 交付前置时间？

交付前置时间（Lead time）指的是从一次提交合并到主分支到它被部署到生产环境所花的时间。这个数字在不同团队之间差异很大，从几周甚至几个月到几分钟都有可能。同样地，在 `$WORK`，我们重视 DORA 的研究，并希望把交付前置时间保持在 10 分钟以内。

要构建一个可靠且交付前置时间出色的系统，需要平衡的测试方法，这通常用[测试金字塔](https://martinfowler.com/articles/practical-test-pyramid.html)来描述。

## 怎么写基本的验收测试

这跟最初的问题有什么关系？我们刚刚写好了一个包，它完全是可以做单元测试的。

正如我提到的，单元测试并不能给我们足够的信心。我们想*真的*确认这个包在与真实运行的程序集成后能工作。我们应当能把之前手动做的检查自动化。

让我们看看那个测试程序：

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// 这种情况通常发生在 ctx 截止前响应没写完，没什么办法
		log.Fatalf("uh oh, didn't shutdown gracefully, some responses may have been lost %v", err)
	}

	// 但愿你看到的总是这一条
	log.Println("shutdown gracefully! all responses were sent")
}
```

你可能已经猜到 `SlowHandler` 里有一个 `time.Sleep` 来延迟响应，这样我才有时间发送 `SIGTERM` 看看会发生什么。其余的部分相当模板化：

- 创建一个 `net/http/Server`；
- 用我们的库把它包起来（参见：[装饰器模式](https://en.wikipedia.org/wiki/Decorator_pattern)）；
- 用包装后的版本来 `ListenAndServe`。

### 验收测试的高层步骤

- 构建程序
- 运行它（并等它在 `8080` 上开始监听）
- 向服务器发送一个 HTTP 请求
- 在服务器有机会发送 HTTP 响应之前，发送 `SIGTERM`
- 看看我们是否还能收到响应

### 构建并运行程序

```go
package acceptancetests

import (
	"fmt"
	"math/rand"
	"net"
	"os"
	"os/exec"
	"path/filepath"
	"syscall"
	"time"
)

const (
	baseBinName = "temp-testbinary"
)

func LaunchTestProgram(port string) (cleanup func(), sendInterrupt func() error, err error) {
	binName, err := buildBinary()
	if err != nil {
		return nil, nil, err
	}

	sendInterrupt, kill, err := runServer(binName, port)

	cleanup = func() {
		if kill != nil {
			kill()
		}
		os.Remove(binName)
	}

	if err != nil {
		cleanup() // 即使监听不正确，程序也可能仍在运行
		return nil, nil, err
	}

	return cleanup, sendInterrupt, nil
}

func buildBinary() (string, error) {
	binName := randomString(10) + "-" + baseBinName

	build := exec.Command("go", "build", "-o", binName)

	if err := build.Run(); err != nil {
		return "", fmt.Errorf("cannot build tool %s: %s", binName, err)
	}
	return binName, nil
}

func runServer(binName string, port string) (sendInterrupt func() error, kill func(), err error) {
	dir, err := os.Getwd()
	if err != nil {
		return nil, nil, err
	}

	cmdPath := filepath.Join(dir, binName)

	cmd := exec.Command(cmdPath)

	if err := cmd.Start(); err != nil {
		return nil, nil, fmt.Errorf("cannot run temp converter: %s", err)
	}

	kill = func() {
		_ = cmd.Process.Kill()
	}

	sendInterrupt = func() error {
		return cmd.Process.Signal(syscall.SIGTERM)
	}

	err = waitForServerListening(port)

	return
}

func waitForServerListening(port string) error {
	for i := 0; i < 30; i++ {
		conn, _ := net.Dial("tcp", net.JoinHostPort("localhost", port))
		if conn != nil {
			conn.Close()
			return nil
		}
		time.Sleep(100 * time.Millisecond)
	}
	return fmt.Errorf("nothing seems to be listening on localhost:%s", port)
}

func randomString(n int) string {
	var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")

	s := make([]rune, n)
	for i := range s {
		s[i] = letters[rand.Intn(len(letters))]
	}
	return string(s)
}
```

`LaunchTestProgram` 负责：
- 构建程序
- 启动程序
- 等待它在 `8080` 端口开始监听
- 提供一个 `cleanup` 函数来杀死并删除该程序，以确保测试结束后我们处于干净的状态
- 提供一个 `interrupt` 函数，向程序发送 `SIGTERM`，让我们能测试相应的行为

不可否认，这并不是世界上最优雅的代码，但你只需要关注导出函数 `LaunchTestProgram`，它调用的那些未导出函数都是无趣的样板代码。

正如前面讨论的，验收测试通常更难搭建。这段代码确实让*测试*代码读起来简洁多了，而且通常验收测试一旦把这些仪式性代码写好，就大功告成了，可以忘掉它了。

### 验收测试本身

我们想为两个程序写两个验收测试，一个有优雅关闭，一个没有，让我们和读者都能看到行为上的差别。有了 `LaunchTestProgram` 来构建并运行程序，给两者写验收测试就很简单了，并且我们能通过一些辅助函数获得复用的好处。

下面是*带*优雅关闭的服务器的测试，[不带优雅关闭的版本可以在 GitHub 找到](https://github.com/quii/go-graceful-shutdown/blob/main/acceptancetests/withoutgracefulshutdown/main_test.go)

```go
package main

import (
	"testing"
	"time"

	"github.com/quii/go-graceful-shutdown/acceptancetests"
	"github.com/quii/go-graceful-shutdown/assert"
)

const (
	port = "8080"
	url  = "<http://localhost:" + port
)

func TestGracefulShutdown(t *testing.T) {
	cleanup, sendInterrupt, err := acceptancetests.LaunchTestProgram(port)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(cleanup)

	// 在我们关掉东西之前，先确认服务器能正常工作
	assert.CanGet(t, url)

	// 发出一个请求，并在它有机会响应之前发送 SIGTERM。
	time.AfterFunc(50*time.Millisecond, func() {
		assert.NoError(t, sendInterrupt())
	})
	// 没有优雅关闭的话，这一行会失败
	assert.CanGet(t, url)

	// 中断之后，服务器应该已关闭，更多请求都不会再工作
	assert.CantGet(t, url)
}
```

把搭建工作封装好之后，测试本身就很全面、能描述行为，而且相对容易理解。

`assert.CanGet/CantGet` 是我做的辅助函数，用来 DRY 化这套测试里这种通用断言。

```go
func CanGet(t testing.TB, url string) {
	errChan := make(chan error)

	go func() {
		res, err := http.Get(url)
		if err != nil {
			errChan <- err
			return
		}
		res.Body.Close()
		errChan <- nil
	}()

	select {
	case err := <-errChan:
		NoError(t, err)
	case <-time.After(3 * time.Second):
		t.Errorf("timed out waiting for request to %q", url)
	}
}
```

它会在一个 goroutine 上向 `URL` 发起一次 `GET`，如果在 3 秒内无错误地响应，就不会失败。`CantGet` 出于篇幅省略了，[但你可以在 GitHub 上看到](https://github.com/quii/go-graceful-shutdown/blob/main/assert/assert.go#L61)。

再次强调，Go 开箱即用就具备了写验收测试所需的所有工具。你*不需要*一个特殊的框架来构建验收测试。

### 小投入，大回报

有了这些测试，读者可以查看示例程序并相信示例*确实*能工作，从而对该包的承诺更有信心。

更重要的是，作为作者，我们获得了**快速反馈**和**巨大的信心**——这个包在真实环境中能工作。

```shell
go test -count=1 ./...
ok  	github.com/quii/go-graceful-shutdown	0.196s
?   	github.com/quii/go-graceful-shutdown/acceptancetests	[no test files]
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withgracefulshutdown	4.785s
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withoutgracefulshutdown	2.914s
?   	github.com/quii/go-graceful-shutdown/assert	[no test files]
```

## 总结

在这篇博文里，我们把验收测试引入了你的测试工具带。当你开始构建真实系统时，它们价值无可替代，是单元测试的重要补充。

*如何*编写验收测试取决于你正在构建的系统，但原则保持不变。把你的系统当作一个"黑盒"。如果你做的是网站，你的测试应当像用户一样行动，所以你会想用一个无头浏览器，比如 [Selenium](https://www.selenium.dev/)，去点击链接、填写表单等等。对于一个 RESTful API，你会用客户端发送 HTTP 请求。

### 在更复杂的系统上更进一步

非平凡的系统通常不会像我们刚讨论的那样是单进程应用。一般来说，你会依赖其他系统，比如数据库。对这些场景，你需要自动化一个本地环境来做测试。像 [docker-compose](https://docs.docker.com/compose/) 这样的工具非常适合启动你需要的容器化本地环境来运行你的系统。

### 下一章

在这篇文章中，验收测试是事后补写的。然而在 [Growing Object-Oriented Software](http://www.growing-object-oriented-software.com) 一书中，作者展示了我们可以用测试驱动的方式使用验收测试，把它当作"北极星"来引导我们的工作。

随着系统变得更复杂，编写和维护验收测试的成本可能会迅速失控。有无数关于开发团队被昂贵的验收测试套件束缚住的故事。

下一章会介绍如何用验收测试引导我们的设计，并提供管理验收测试成本的原则与技术。

### 提升开源项目的质量

如果你在写打算分享出去的包，我建议你创建
简单的示例程序来展示你的包做了什么，并花时间编写易于理解的验收测试，给自己以及你工作的潜在用户一份信心。

就像[可测试示例](https://go.dev/blog/examples)一样，在开发者体验上多投入这一点点，对于建立人们对你工作的信任、降低你自己的维护成本而言都大有帮助。

## `$WORK` 的招聘小广告

如果你想在这样一个环境里工作——同其他工程师一起解决有趣的问题、住在伦敦或波尔图附近、并且喜欢本章和本书的内容——请[在 Twitter 上联系我](https://twitter.com/quii)，也许我们很快就能一起共事！
