# Sync

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/sync)**

我们想做一个在并发环境下安全使用的计数器。

我们会先从一个不安全的计数器开始，并验证它在单线程环境下能正常工作。

然后我们会通过一个测试，让多个 goroutine 同时使用这个计数器，把它的"不安全"暴露出来，再修复它。

## 先写测试

我们希望 API 提供一个方法用来递增计数器，以及一个方法用来获取它的值。

```go
func TestCounter(t *testing.T) {
	t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
		counter := Counter{}
		counter.Inc()
		counter.Inc()
		counter.Inc()

		if counter.Value() != 3 {
			t.Errorf("got %d, want %d", counter.Value(), 3)
		}
	})
}
```

## 试着运行测试

```
./sync_test.go:9:14: undefined: Counter
```

## 写最少量的代码让测试运行起来，并检查失败的输出

我们来定义 `Counter`。

```go
type Counter struct {
}
```

再试一次，会得到下面的失败

```
./sync_test.go:14:10: counter.Inc undefined (type Counter has no field or method Inc)
./sync_test.go:18:13: counter.Value undefined (type Counter has no field or method Value)
```

为了让测试最终能跑起来，我们定义这些方法

```go
func (c *Counter) Inc() {

}

func (c *Counter) Value() int {
	return 0
}
```

它现在应该能跑起来并失败

```
=== RUN   TestCounter
=== RUN   TestCounter/incrementing_the_counter_3_times_leaves_it_at_3
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/incrementing_the_counter_3_times_leaves_it_at_3 (0.00s)
    	sync_test.go:27: got 0, want 3
```

## 写足够的代码让测试通过

对像我们这样的 Go 专家来说这应该是小菜一碟。我们需要在数据类型里保存一些状态，然后每次调用 `Inc` 时递增它

```go
type Counter struct {
	value int
}

func (c *Counter) Inc() {
	c.value++
}

func (c *Counter) Value() int {
	return c.value
}
```

## 重构

可重构的不多，但既然我们要围绕 `Counter` 写更多测试，我们写一个小的断言函数 `assertCount`，让测试读起来更清晰一些。

```go
t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
	counter := Counter{}
	counter.Inc()
	counter.Inc()
	counter.Inc()

	assertCounter(t, counter, 3)
})
```
```go
func assertCounter(t testing.TB, got Counter, want int) {
	t.Helper()
	if got.Value() != want {
		t.Errorf("got %d, want %d", got.Value(), want)
	}
}
```

## 下一步

那很简单，但现在我们有了一个新需求：它必须在并发环境下安全使用。我们需要写一个失败的测试来验证这一点。

## 先写测试

```go
t.Run("it runs safely concurrently", func(t *testing.T) {
	wantedCount := 1000
	counter := Counter{}

	var wg sync.WaitGroup
	wg.Add(wantedCount)

	for i := 0; i < wantedCount; i++ {
		go func() {
			counter.Inc()
			wg.Done()
		}()
	}
	wg.Wait()

	assertCounter(t, counter, wantedCount)
})
```

这会循环 `wantedCount` 次，每次启动一个 goroutine 来调用 `counter.Inc()`。

我们使用了 [`sync.WaitGroup`](https://golang.org/pkg/sync/#WaitGroup)，它是同步并发流程的一种便利方式。

> WaitGroup 等待一组 goroutine 完成。主 goroutine 调用 Add 来设置要等待的 goroutine 数量。然后每个 goroutine 运行，完成时调用 Done。与此同时，可以用 Wait 来阻塞，直到所有 goroutine 都完成。

通过先等 `wg.Wait()` 完成再做断言，我们可以确保所有 goroutine 都尝试过 `Inc` 这个 `Counter`。

## 试着运行测试

```
=== RUN   TestCounter/it_runs_safely_in_a_concurrent_envionment
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/it_runs_safely_in_a_concurrent_envionment (0.00s)
    	sync_test.go:26: got 939, want 1000
FAIL
```

测试 _很可能_ 会以一个不同的数字失败，但无论如何，它都说明了当多个 goroutine 同时尝试修改计数器的值时它无法工作。

## 写足够的代码让测试通过

一个简单的方案是给 `Counter` 加一把锁，保证一次只有一个 goroutine 能递增计数器。Go 的 [`Mutex`](https://golang.org/pkg/sync/#Mutex) 提供了这种锁：

> Mutex 是一把互斥锁。Mutex 的零值是未上锁状态。

```go
type Counter struct {
	mu    sync.Mutex
	value int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}
```

这意味着任何调用 `Inc` 的 goroutine，如果它先到，就会拿到 `Counter` 的锁。其他 goroutine 必须等它 `Unlock` 后才能进入。

如果你现在重新运行测试，它应该能通过，因为每个 goroutine 都得排队等候才能改变状态。

## 我看到过别的例子里 `sync.Mutex` 是嵌入到结构体里的。

你可能见过这样的例子

```go
type Counter struct {
	sync.Mutex
	value int
}
```

可以争辩说这能让代码看起来更优雅一点。

```go
func (c *Counter) Inc() {
	c.Lock()
	defer c.Unlock()
	c.value++
}
```

这 _看起来_ 不错，但虽然编程是一门高度主观的学问，这种做法 **既糟糕又错误**。

人们有时会忘记，类型嵌入意味着该类型的方法变成了 _公开接口的一部分_；而你通常并不希望这样。记住，我们对公开 API 应该格外谨慎，一旦把某个东西公开出去，别的代码就可能耦合到它上面。我们要尽量避免不必要的耦合。

把 `Lock` 和 `Unlock` 暴露出去往好里说会让人困惑，往坏里说，如果你的类型的调用者开始去调这些方法，可能会对你的软件造成严重伤害。

![Showing how a user of this API can wrongly change the state of the lock](https://i.imgur.com/SWYNpwm.png)

_这看起来真是个糟糕的主意_

## 复制 mutex

我们的测试通过了，但代码仍有点危险。

如果你对代码运行 `go vet`，应该会得到类似下面的错误

```
sync/v2/sync_test.go:16: call of assertCounter copies lock value: v1.Counter contains sync.Mutex
sync/v2/sync_test.go:39: assertCounter passes lock by value: v1.Counter contains sync.Mutex
```

看一下 [`sync.Mutex`](https://golang.org/pkg/sync/#Mutex) 的文档就知道为什么了

> A Mutex must not be copied after first use.

当我们把 `Counter`（按值）传给 `assertCounter` 时，它会尝试创建 mutex 的副本。

为了解决这个问题，我们应当传入一个指向 `Counter` 的指针，所以修改 `assertCounter` 的签名

```go
func assertCounter(t testing.TB, got *Counter, want int)
```

我们的测试将无法编译，因为我们传入的是 `Counter` 而不是 `*Counter`。要解决它，我倾向于创建一个构造函数，向 API 的使用者表明最好不要自己初始化这个类型。

```go
func NewCounter() *Counter {
	return &Counter{}
}
```

在测试中初始化 `Counter` 时使用这个函数。

## 总结

我们涉及了 [sync 包](https://golang.org/pkg/sync/) 里的几样东西

- `Mutex` 让我们能给数据加锁
- `WaitGroup` 是等待 goroutine 完成任务的方式

### 何时使用锁，何时使用 channel 和 goroutine？

[我们之前在第一篇并发章节里讲过 goroutine](concurrency.md)，它让我们能写出安全的并发代码，那为什么还要用锁呢？
[Go wiki 有一个专门讨论这个话题的页面：Mutex Or Channel](https://go.dev/wiki/MutexOrChannel)

> Go 新手常犯的一个错误是仅仅因为"可以"或者"好玩"就过度使用 channel 和 goroutine。如果 sync.Mutex 最适合你的问题，别怕用它。Go 在工具选择上很务实，让你用最能解决问题的工具，而不强迫你写成某一种风格。

简单概括一下：

- **传递数据所有权时使用 channel**
- **管理状态时使用 mutex**

### go vet

记得在你的构建脚本里使用 go vet，它能在那些隐蔽的 bug 影响到可怜的用户之前提醒你。

### 不要因为方便就用嵌入

- 想想嵌入对你公开 API 的影响。
- 你 _真的_ 想把这些方法暴露出去，让别人把代码耦合到它们上面吗？
- 就 mutex 而言，这可能造成极不可预测、奇形怪状的灾难性后果，想象一下某段恶意代码在不该解锁时解了 mutex 的锁，这会引发一些非常奇怪、难以追踪的 bug。
