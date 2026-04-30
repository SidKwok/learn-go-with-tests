# Select

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/select)**

你被要求实现一个名为 `WebsiteRacer` 的函数，它接收两个 URL，对它们发起 HTTP GET 请求来"赛跑"，返回先响应的那个 URL。如果 10 秒内都没有任何一个返回，应该返回一个 `error`。

为此，我们将使用：

- `net/http` 来发起 HTTP 调用。
- `net/http/httptest` 帮我们做测试。
- goroutine。
- `select` 来同步进程。

## 先写测试

我们从一些朴素的方式入手开始。

```go
func TestRacer(t *testing.T) {
	slowURL := "http://www.facebook.com"
	fastURL := "http://www.quii.dev"

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

我们知道这并不完美，存在一些问题，但这是一个起点。重要的是不要太执着于一开始就把事情做到完美。

## 尝试运行测试

`./racer_test.go:14:9: undefined: Racer`

## 写最少的代码让测试能跑起来，并检查失败的测试输出

```go
func Racer(a, b string) (winner string) {
	return
}
```

`racer_test.go:25: got '', want 'http://www.quii.dev'`

## 写足够的代码让它通过

```go
func Racer(a, b string) (winner string) {
	startA := time.Now()
	http.Get(a)
	aDuration := time.Since(startA)

	startB := time.Now()
	http.Get(b)
	bDuration := time.Since(startB)

	if aDuration < bDuration {
		return a
	}

	return b
}
```

对于每个 URL：

1. 我们用 `time.Now()` 记录在尝试请求 `URL` 之前的时间。
1. 然后用 [`http.Get`](https://golang.org/pkg/net/http/#Client.Get) 对 `URL` 发起一个 HTTP `GET` 请求。这个函数返回一个 [`http.Response`](https://golang.org/pkg/net/http/#Response) 和一个 `error`，但目前我们对这些值还不感兴趣。
1. `time.Since` 接受开始时间，返回两者差值的 `time.Duration`。

做完这些之后，我们只需要比较两个时长，看哪个更快。

### 问题

测试结果可能让你通过，也可能不会。问题在于我们正在访问真实的网站来测试我们自己的逻辑。

测试使用 HTTP 的代码非常常见，所以 Go 在标准库中提供了帮你测试它的工具。

在 mock 和依赖注入的章节里，我们讲过：理想情况下，我们不希望依赖外部服务来测试自己的代码，因为它们可能：

- 慢
- 不稳定
- 没办法测边界情况

标准库里有一个叫做 [`net/http/httptest`](https://golang.org/pkg/net/http/httptest/) 的包，让用户可以轻松创建一个 mock HTTP 服务器。

让我们改一下测试，用 mock 来获得可靠且可控的服务器进行测试。

```go
func TestRacer(t *testing.T) {

	slowServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(20 * time.Millisecond)
		w.WriteHeader(http.StatusOK)
	}))

	fastServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}

	slowServer.Close()
	fastServer.Close()
}
```

语法看起来可能有点繁琐，但慢慢看就好。

`httptest.NewServer` 接受一个 `http.HandlerFunc`，我们通过一个 _匿名函数_ 把它传进去。

`http.HandlerFunc` 是一个类型，长得像这样：`type HandlerFunc func(ResponseWriter, *Request)`。

它真正在表达的就是：它需要一个接受 `ResponseWriter` 和 `Request` 的函数，对一个 HTTP 服务器来说这并不令人意外。

事实上这里没有什么额外的魔法，**这也是你在 Go 中编写一个 _真实_ HTTP 服务器的方式**。唯一的区别是我们用 `httptest.NewServer` 把它包了一层，这样在测试中使用起来更方便：它会找一个空闲端口来监听，测试结束后你可以关闭它。

在我们的两个服务器内部，我们让慢的那个在收到请求时执行一个短暂的 `time.Sleep`，这样它就比另一个慢。两个服务器都会通过 `w.WriteHeader(http.StatusOK)` 返回一个 `OK` 响应给调用方。

如果你重新运行测试，它现在肯定会通过，并且应该更快了。可以试着调一下这些 sleep，故意让测试失败看看。

## 重构

我们的生产代码和测试代码里都有一些重复。

```go
func Racer(a, b string) (winner string) {
	aDuration := measureResponseTime(a)
	bDuration := measureResponseTime(b)

	if aDuration < bDuration {
		return a
	}

	return b
}

func measureResponseTime(url string) time.Duration {
	start := time.Now()
	http.Get(url)
	return time.Since(start)
}
```

这种 DRY 处理让我们的 `Racer` 代码更易读。

```go
func TestRacer(t *testing.T) {

	slowServer := makeDelayedServer(20 * time.Millisecond)
	fastServer := makeDelayedServer(0 * time.Millisecond)

	defer slowServer.Close()
	defer fastServer.Close()

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

func makeDelayedServer(delay time.Duration) *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(delay)
		w.WriteHeader(http.StatusOK)
	}))
}
```

我们把创建假服务器的逻辑重构成了一个名为 `makeDelayedServer` 的函数，把一些不重要的代码从测试里挪出去，并减少重复。

### `defer`

通过在函数调用前加上 `defer`，它会在 _所在函数结束时_ 调用那个函数。

有时候你需要清理资源，比如关闭一个文件，或者在我们的例子里，关闭一个服务器，这样它就不会继续监听端口。

你希望它在函数结束时执行，但又希望让指令靠近创建服务器的地方，方便后来阅读代码的人理解。

我们的重构是个改进，并且就目前覆盖到的 Go 特性来说算是一个合理的方案，但我们可以让方案更简单。

### 同步进程

- 当 Go 这么擅长并发的时候，我们为什么要一个接一个地测试网站速度？我们应该能同时检查两个。
- 我们其实并不关心请求的 _确切响应时间_，我们只想知道哪个先回来。

为了做到这一点，我们要引入一个新的语法结构 `select`，它能帮我们非常简单清晰地同步进程。

```go
func Racer(a, b string) (winner string) {
	select {
	case <-ping(a):
		return a
	case <-ping(b):
		return b
	}
}

func ping(url string) chan struct{} {
	ch := make(chan struct{})
	go func() {
		http.Get(url)
		close(ch)
	}()
	return ch
}
```

#### `ping`

我们定义了一个 `ping` 函数，它创建一个 `chan struct{}` 并返回它。

在我们这里，我们 _不关心_ 往 channel 里发送的是什么类型，_我们只是想发出一个完成信号_，关闭这个 channel 完美地达成了这个目的！

为什么用 `struct{}` 而不是其他类型，比如 `bool`？因为从内存角度看 `chan struct{}` 是最小的可用数据类型，相比 `bool`，它不会有内存分配。既然我们是关闭 channel 而不是往里发送任何东西，那为什么要分配任何东西呢？

在同一个函数内部，我们启动了一个 goroutine，它会在 `http.Get(url)` 完成之后向那个 channel 发送一个信号。

##### 总是用 `make` 创建 channel

注意我们在创建 channel 时必须使用 `make`，而不是写 `var ch chan struct{}`。当你使用 `var` 时，变量会被初始化为该类型的"零值"。所以对于 `string` 是 `""`，对于 `int` 是 0，等等。

对于 channel，零值是 `nil`，如果你试图用 `<-` 向它发送，会永远阻塞，因为你不能向 `nil` 的 channel 发送。

[你可以在 Go Playground 中实际看到这一点](https://play.golang.org/p/IIbeAox5jKA)
#### `select`

你应该还记得在并发那一章里，你可以用 `myVar := <-ch` 等待一个值被发送到 channel。这是一个 _阻塞_ 调用，因为你在等一个值。

`select` 让你可以在 _多个_ channel 上等待。第一个发送值过来的"获胜"，对应 `case` 下的代码会被执行。

我们在 `select` 中使用 `ping` 来设置两个 channel，每个 `URL` 对应一个。先写到自己 channel 的那个，会让它在 `select` 中对应的代码被执行，结果就是它的 `URL` 被返回（成为获胜者）。

经过这些修改，我们代码背后的意图非常清晰了，而且实现也确实更简单了。

### 超时

我们最后一个需求是：如果 `Racer` 用时超过 10 秒，要返回一个错误。

## 先写测试

```go
func TestRacer(t *testing.T) {
	t.Run("compares speeds of servers, returning the url of the fastest one", func(t *testing.T) {
		slowServer := makeDelayedServer(20 * time.Millisecond)
		fastServer := makeDelayedServer(0 * time.Millisecond)

		defer slowServer.Close()
		defer fastServer.Close()

		slowURL := slowServer.URL
		fastURL := fastServer.URL

		want := fastURL
		got, _ := Racer(slowURL, fastURL)

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("returns an error if a server doesn't respond within 10s", func(t *testing.T) {
		serverA := makeDelayedServer(11 * time.Second)
		serverB := makeDelayedServer(12 * time.Second)

		defer serverA.Close()
		defer serverB.Close()

		_, err := Racer(serverA.URL, serverB.URL)

		if err == nil {
			t.Error("expected an error but didn't get one")
		}
	})
}
```

我们让测试服务器超过 10 秒才返回，以验证这个场景，并期望 `Racer` 现在返回两个值：获胜的 URL（在这个测试里我们用 `_` 忽略它）和一个 `error`。

注意我们在原先的测试里也处理了 error 的返回值，我们暂时用 `_` 来确保测试能跑起来。

## 尝试运行测试

`./racer_test.go:37:10: assignment mismatch: 2 variables but Racer returns 1 value`

## 写最少的代码让测试能跑起来，并检查失败的测试输出

```go
func Racer(a, b string) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	}
}
```

修改 `Racer` 的签名，让它返回获胜者和一个 `error`。在正常情况下返回 `nil`。

编译器会抱怨你的 _第一个测试_ 只接收一个返回值，所以把那一行改成 `got, err := Racer(slowURL, fastURL)`，并知道我们应该检查在正常场景下我们 _不会_ 收到错误。

如果你现在运行它，11 秒后它会失败。

```
--- FAIL: TestRacer (12.00s)
    --- FAIL: TestRacer/returns_an_error_if_a_server_doesn't_respond_within_10s (12.00s)
        racer_test.go:40: expected an error but didn't get one
```

## 写足够的代码让它通过

```go
func Racer(a, b string) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(10 * time.Second):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

`time.After` 在使用 `select` 时是一个非常方便的函数。虽然在我们这里不会发生，但你完全有可能写出永远阻塞的代码，如果你监听的 channel 永远不返回值的话。`time.After` 返回一个 `chan`（和 `ping` 类似），并在你定义的时间之后向其发送一个信号。

对我们来说这很完美：如果 `a` 或 `b` 成功返回，它们获胜，但如果时间到了 10 秒，`time.After` 就会发送信号，我们会返回一个 `error`。

### 慢测试

我们碰到的问题是：这个测试要跑 10 秒。对这么简单的一段逻辑来说，感觉不太好。

我们可以做的是把超时变成可配置的。这样在测试中我们可以使用一个非常短的超时，而当代码在真实世界中使用时则可以设为 10 秒。

```go
func Racer(a, b string, timeout time.Duration) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

我们的测试现在编译不通过，因为我们没有提供 timeout。

在着急把这个默认值加进我们两个测试之前，我们先 _听听测试在说什么_。

- 我们在"正常路径"测试中关心 timeout 吗？
- 需求里明确提到了 timeout。

基于这个认知，我们做一点重构，既照顾我们的测试，也照顾我们代码的使用者。

```go
var tenSecondTimeout = 10 * time.Second

func Racer(a, b string) (winner string, error error) {
	return ConfigurableRacer(a, b, tenSecondTimeout)
}

func ConfigurableRacer(a, b string, timeout time.Duration) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

我们的用户和第一个测试可以使用 `Racer`（底层调用 `ConfigurableRacer`），而我们处理失败路径的测试可以直接使用 `ConfigurableRacer`。

```go
func TestRacer(t *testing.T) {

	t.Run("compares speeds of servers, returning the url of the fastest one", func(t *testing.T) {
		slowServer := makeDelayedServer(20 * time.Millisecond)
		fastServer := makeDelayedServer(0 * time.Millisecond)

		defer slowServer.Close()
		defer fastServer.Close()

		slowURL := slowServer.URL
		fastURL := fastServer.URL

		want := fastURL
		got, err := Racer(slowURL, fastURL)

		if err != nil {
			t.Fatalf("did not expect an error but got one %v", err)
		}

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("returns an error if a server doesn't respond within the specified time", func(t *testing.T) {
		server := makeDelayedServer(25 * time.Millisecond)

		defer server.Close()

		_, err := ConfigurableRacer(server.URL, server.URL, 20*time.Millisecond)

		if err == nil {
			t.Error("expected an error but didn't get one")
		}
	})
}
```

我在第一个测试里加了一个最后的检查，验证我们没有收到 `error`。

## 总结

### `select`

- 帮你在多个 channel 上等待。
- 有时候你会想在某一个 `case` 中包含 `time.After`，以防止你的系统永远阻塞。

### `httptest`

- 一种创建测试服务器的便捷方式，让你的测试可靠且可控。
- 使用与"真正的" `net/http` 服务器相同的接口，保持一致性，你需要学习的东西也更少。
