# Context

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/context)**

软件经常会启动一些长时间运行、资源密集的进程（通常在 goroutine 中）。如果触发它的动作被取消或因某种原因失败，你需要在你的应用中以一致的方式停止这些进程。

如果你不管理它，你那个让你引以为傲的灵活的 Go 应用可能会开始出现难以调试的性能问题。

本章我们将使用 `context` 包来帮助我们管理长时间运行的进程。

我们从一个经典的例子开始：一个 web 服务器，被调用时会启动一个可能长时间运行的进程来获取一些数据，作为响应返回。

我们将演练这样一个场景：用户在数据被取回之前取消了请求，我们要确保进程被通知放弃工作。

我已经为我们准备了一些走 happy path 的代码作为开始。下面是我们的服务器代码。

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, store.Fetch())
	}
}
```

`Server` 函数接受一个 `Store` 并返回给我们一个 `http.HandlerFunc`。Store 的定义如下：

```go
type Store interface {
	Fetch() string
}
```

返回的函数调用 `store` 的 `Fetch` 方法获取数据并写入响应。

我们有一个对应的 `Store` 的 spy，我们在测试中使用它。

```go
type SpyStore struct {
	response string
}

func (s *SpyStore) Fetch() string {
	return s.response
}

func TestServer(t *testing.T) {
	data := "hello, world"
	svr := Server(&SpyStore{data})

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}
}
```

现在我们有了 happy path，我们想做一个更现实的场景：`Store` 还没完成 `Fetch`，用户就取消了请求。

## 先写测试

我们的 handler 需要一种方式告诉 `Store` 取消工作，所以更新接口。

```go
type Store interface {
	Fetch() string
	Cancel()
}
```

我们需要调整 spy，让它返回 `data` 需要花一些时间，并提供一种方式来确认它被通知取消了。它必须把 `Cancel` 加为方法以实现 `Store` 接口。

```go
type SpyStore struct {
	response  string
	cancelled bool
}

func (s *SpyStore) Fetch() string {
	time.Sleep(100 * time.Millisecond)
	return s.response
}

func (s *SpyStore) Cancel() {
	s.cancelled = true
}
```

让我们加一个新测试，在 100 毫秒之前取消请求，并检查 store 是否被取消。

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)

	cancellingCtx, cancel := context.WithCancel(request.Context())
	time.AfterFunc(5*time.Millisecond, cancel)
	request = request.WithContext(cancellingCtx)

	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if !store.cancelled {
		t.Error("store was not told to cancel")
	}
})
```

来自 [Go 博客：Context](https://blog.golang.org/context)

> context 包提供了从已有 Context 派生新 Context 值的函数。这些值形成一棵树：当一个 Context 被取消时，从它派生的所有 Context 也都被取消。

重要的是你要派生你的 context，以便在给定请求的整个调用栈中传播取消。

我们做的是从我们的 `request` 派生一个新的 `cancellingCtx`，它返回给我们一个 `cancel` 函数。然后我们用 `time.AfterFunc` 安排该函数在 5 毫秒后被调用。最后我们通过调用 `request.WithContext` 在请求中使用这个新的 context。

## 试着运行测试

测试如预期失败。

```
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.00s)
    	context_test.go:62: store was not told to cancel
```

## 写足够的代码让它通过

记得对 TDD 保持纪律。写 _最小量_ 的代码让我们的测试通过。

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		store.Cancel()
		fmt.Fprint(w, store.Fetch())
	}
}
```

这让这个测试通过了，但感觉不好对吧！我们当然不应该 _在每个请求_ 都在 fetch 之前调用 `Cancel()`。

通过保持纪律，它暴露了我们测试中的一个缺陷，这是好事！

我们需要更新 happy path 测试，断言它没有被取消。

```go
t.Run("returns data from store", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}

	if store.cancelled {
		t.Error("it should not have cancelled the store")
	}
})
```

跑两个测试，happy path 测试现在应该失败了，于是我们被迫做一个更合理的实现。

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		data := make(chan string, 1)

		go func() {
			data <- store.Fetch()
		}()

		select {
		case d := <-data:
			fmt.Fprint(w, d)
		case <-ctx.Done():
			store.Cancel()
		}
	}
}
```

我们在这里做了什么？

`context` 有一个 `Done()` 方法，它返回一个 channel，当 context 被"完成"或"取消"时会收到一个信号。我们想监听那个信号，如果收到就调用 `store.Cancel`，但如果我们的 `Store` 在那之前完成了 `Fetch`，我们就忽略它。

为了管理这一点，我们在 goroutine 中运行 `Fetch`，它会把结果写入新 channel `data`。然后我们用 `select` 让两个异步过程"赛跑"，然后我们要么写一个响应，要么 `Cancel`。

## 重构

我们可以稍微重构一下测试代码，给 spy 加上断言方法

```go
type SpyStore struct {
	response  string
	cancelled bool
	t         *testing.T
}

func (s *SpyStore) assertWasCancelled() {
	s.t.Helper()
	if !s.cancelled {
		s.t.Error("store was not told to cancel")
	}
}

func (s *SpyStore) assertWasNotCancelled() {
	s.t.Helper()
	if s.cancelled {
		s.t.Error("store was told to cancel")
	}
}
```

记得在创建 spy 时传入 `*testing.T`。

```go
func TestServer(t *testing.T) {
	data := "hello, world"

	t.Run("returns data from store", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)
		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		if response.Body.String() != data {
			t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
		}

		store.assertWasNotCancelled()
	})

	t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)

		cancellingCtx, cancel := context.WithCancel(request.Context())
		time.AfterFunc(5*time.Millisecond, cancel)
		request = request.WithContext(cancellingCtx)

		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		store.assertWasCancelled()
	})
}
```

这种方式还行，但它符合惯用法吗？

让我们的 web 服务器去关心手动取消 `Store` 是合理的吗？如果 `Store` 也碰巧依赖其他慢运行的进程怎么办？我们就得确保 `Store.Cancel` 把取消正确传播给它所有的依赖。

`context` 的主要要点之一就是它是一种提供取消的一致方式。

[来自 go doc](https://golang.org/pkg/context/)

> 服务器收到的入站请求应当创建一个 Context，向服务器发出的出站调用应当接受一个 Context。两者之间的函数调用链必须传播该 Context，并可选择用 WithCancel、WithDeadline、WithTimeout 或 WithValue 派生出新的 Context 替换它。当一个 Context 被取消时，从它派生的所有 Context 也被取消。

再次来自 [Go 博客：Context](https://blog.golang.org/context)：

> 在 Google，我们要求 Go 程序员把 Context 参数作为入站和出站请求之间调用路径上每个函数的第一个参数。这让由许多不同团队开发的 Go 代码能够良好地互操作。它提供了对超时和取消的简单控制，并确保了像安全凭证这样关键的值能在 Go 程序中正确传递。

（暂停一下，想想每个函数都得传一个 context 的影响，以及它在使用上的体验。）

感觉有点不安？很好。不过让我们尝试遵循这种方式，把 `context` 传给我们的 `Store`，让它来负责。这样它也可以把 `context` 传给它的依赖，这些依赖也可以负责让自己停止。

## 先写测试

我们必须改我们已有的测试，因为它们的职责正在改变。我们的 handler 现在唯一的职责是确保把 context 传给下游的 `Store`，并处理 `Store` 在被取消时返回的错误。

让我们更新 `Store` 接口，反映新的职责。

```go
type Store interface {
	Fetch(ctx context.Context) (string, error)
}
```

暂时删掉 handler 内的代码

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
	}
}
```

更新我们的 `SpyStore`

```go
type SpyStore struct {
	response string
	t        *testing.T
}

func (s *SpyStore) Fetch(ctx context.Context) (string, error) {
	data := make(chan string, 1)

	go func() {
		var result string
		for _, c := range s.response {
			select {
			case <-ctx.Done():
				log.Println("spy store got cancelled")
				return
			default:
				time.Sleep(10 * time.Millisecond)
				result += string(c)
			}
		}
		data <- result
	}()

	select {
	case <-ctx.Done():
		return "", ctx.Err()
	case res := <-data:
		return res, nil
	}
}
```

我们必须让我们的 spy 像一个真正与 `context` 协作的方法那样行事。

我们在模拟一个慢进程，在一个 goroutine 中通过逐字符追加字符串来缓慢地构建结果。当 goroutine 完成它的工作时，把字符串写入 `data` channel。goroutine 监听 `ctx.Done`，如果在那个 channel 中收到信号就停止工作。

最后，代码用另一个 `select` 等待该 goroutine 完成它的工作，或等取消发生。

这与我们之前的方式类似，我们用 Go 的并发原语让两个异步过程相互赛跑，决定我们返回什么。

当你写自己接受 `context` 的函数和方法时，会采用类似的方法，所以确保你理解这是怎么回事。

最后我们可以更新我们的测试。把取消测试注释掉，先修 happy path 测试。

```go
t.Run("returns data from store", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data, t: t}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}
})
```

## 试着运行测试

```
=== RUN   TestServer/returns_data_from_store
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/returns_data_from_store (0.00s)
    	context_test.go:22: got "", want "hello, world"
```

## 写足够的代码让它通过

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, _ := store.Fetch(r.Context())
		fmt.Fprint(w, data)
	}
}
```

我们的 happy path 应该……开心了。现在我们可以修另一个测试。

## 先写测试

我们需要测试在错误情况下我们不写任何响应。可惜 `httptest.ResponseRecorder` 没有方式弄清这一点，所以我们必须自己写一个 spy 来测试。

```go
type SpyResponseWriter struct {
	written bool
}

func (s *SpyResponseWriter) Header() http.Header {
	s.written = true
	return nil
}

func (s *SpyResponseWriter) Write([]byte) (int, error) {
	s.written = true
	return 0, errors.New("not implemented")
}

func (s *SpyResponseWriter) WriteHeader(statusCode int) {
	s.written = true
}
```

我们的 `SpyResponseWriter` 实现了 `http.ResponseWriter`，所以我们可以在测试中使用它。

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data, t: t}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)

	cancellingCtx, cancel := context.WithCancel(request.Context())
	time.AfterFunc(5*time.Millisecond, cancel)
	request = request.WithContext(cancellingCtx)

	response := &SpyResponseWriter{}

	svr.ServeHTTP(response, request)

	if response.written {
		t.Error("a response should not have been written")
	}
})
```

## 试着运行测试

```
=== RUN   TestServer
=== RUN   TestServer/tells_store_to_cancel_work_if_request_is_cancelled
--- FAIL: TestServer (0.01s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.01s)
    	context_test.go:47: a response should not have been written
```

## 写足够的代码让它通过

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, err := store.Fetch(r.Context())

		if err != nil {
			return // todo: 按你喜欢的方式记录错误
		}

		fmt.Fprint(w, data)
	}
}
```

我们可以看到，做完这一步后服务器代码已经简化了，因为它不再显式地负责取消，它只是把 `context` 传下去，依赖下游函数尊重可能发生的任何取消。

## 总结

### 我们涵盖了什么

- 如何测试一个被客户端取消请求的 HTTP handler。
- 如何使用 context 来管理取消。
- 如何写一个接受 `context` 并使用 goroutine、`select` 和 channel 来取消自身的函数。
- 遵循 Google 的指南：通过在调用栈中传播请求范围的 context 来管理取消。
- 必要时如何为 `http.ResponseWriter` 写自己的 spy。

### 那 context.Value 怎么样？

[Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/) 和我有相似的看法。

> 如果你在我（不存在的）公司里使用 ctx.Value，你就被开除了

一些工程师主张通过 `context` 传值，因为这 _感觉方便_。

方便往往是糟糕代码的根源。

`context.Values` 的问题是它只是一个无类型的 map，所以你没有类型安全，而且你必须处理它实际上不包含你的值的情况。你必须在一个模块和另一个模块之间制造 map key 的耦合，如果有人改了什么东西，事情就开始坏掉。

简而言之，**如果一个函数需要某些值，把它们作为带类型的参数传入，而不是试图从 `context.Value` 拿**。这样就能在静态层面被检查，并对所有人都有文档说明。

#### 但是……

另一方面，把与请求正交的信息（比如 trace id）放在 context 中可能很有用。这些信息可能不会被你调用栈中的每个函数需要，要把它们都放进函数签名会让签名非常凌乱。

[Jack Lindamood 说 **Context.Value 应当告知，而不是控制**](https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39)

> context.Value 的内容是给维护者的，而不是给使用者的。它绝不应该是文档化或预期结果的必需输入。

### 进阶材料

- 我非常喜欢读 [Michal Štrba 的《Context should go away for Go 2》](https://faiface.github.io/post/context-should-go-away-go2/)。他的论点是，必须到处传 `context` 是一种坏味道，它指向了 Go 在取消方面的语言层面缺陷。他说如果这能在语言层面解决而不是在库层面解决会更好。在那之前，如果你想管理长时间运行的进程，你会需要 `context`。
- [Go 博客进一步描述了使用 `context` 的动机，并有一些示例](https://blog.golang.org/context)
