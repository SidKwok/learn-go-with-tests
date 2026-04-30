# 感知 context 的 reader

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/context-aware-reader)**

本章演示了如何用测试驱动开发的方式实现一个感知 context 的 `io.Reader`，它由 Mat Ryer 和 David Hernandez 在[The Pace Dev Blog](https://pace.dev/blog/2020/02/03/context-aware-ioreader-for-golang-by-mat-ryer)上写过。

## 感知 context 的 reader？

首先，简单介绍一下 `io.Reader`。

如果你读过本书的其他章节，会在我们打开文件、编码 JSON 以及做各种其他常见任务时遇到过 `io.Reader`。它是一个对从 _某个东西_ 读取数据的简单抽象

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

通过使用 `io.Reader`，你可以从标准库中获得大量复用，它是一个非常常用的抽象（连同它的对应物 `io.Writer`）

### 感知 context？

[在前面的章节](context.md)中，我们讨论了如何用 `context` 来提供取消。如果你正在执行可能耗费大量计算资源的任务，并且希望能停下它们，这特别有用。

当你使用 `io.Reader` 时，你对速度没有保证，可能 1 纳秒，也可能上百小时。在你自己的应用中能取消这种任务可能很有用，这正是 Mat 和 David 写的内容。

他们结合了两个简单的抽象（`context.Context` 和 `io.Reader`）来解决这个问题。

让我们试着用 TDD 来开发一些功能，让我们能包装一个 `io.Reader`，使其可以被取消。

测试这个有一个有趣的挑战。通常使用 `io.Reader` 时，你是把它提供给某个其他函数，并不真的关心细节；比如 `json.NewDecoder` 或 `io.ReadAll`。

我们想演示的是这样的场景

> 给一个内容为 "ABCDEF" 的 `io.Reader`，当我在中途发送取消信号，然后再尝试继续读时，我什么都得不到，所以我得到的只有 "ABC"

我们再看一眼这个接口。

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

`Reader` 的 `Read` 方法会把它拥有的内容读到我们提供的 `[]byte` 中。

所以，与其一次读完所有内容，我们可以：

 - 提供一个固定大小的、装不下所有内容的字节数组
 - 发送一个取消信号
 - 再尝试读，这次应该返回一个错误且读到 0 字节

现在，先写一个"happy path"测试，里面没有取消，这样我们可以在还不需要写生产代码之前先熟悉问题。

```go
func TestContextAwareReader(t *testing.T) {
	t.Run("lets just see how a normal reader works", func(t *testing.T) {
		rdr := strings.NewReader("123456")
		got := make([]byte, 3)
		_, err := rdr.Read(got)

		if err != nil {
			t.Fatal(err)
		}

		assertBufferHas(t, got, "123")

		_, err = rdr.Read(got)

		if err != nil {
			t.Fatal(err)
		}

		assertBufferHas(t, got, "456")
	})
}

func assertBufferHas(t testing.TB, buf []byte, want string) {
	t.Helper()
	got := string(buf)
	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

- 用一个有数据的字符串创建一个 `io.Reader`
- 一个比 reader 内容还小的字节数组用来读入
- 调用 read，检查内容，重复。

由此我们可以想象在第二次读之前发送某种取消信号来改变行为。

现在我们看了它怎么工作，接下来用 TDD 实现剩下的功能。

## 先写测试

我们想能把一个 `io.Reader` 和一个 `context.Context` 组合起来。

用 TDD 时，最好从想象你期望的 API 开始，并为它写一个测试。

由此让编译器和失败的测试输出引导我们走向解决方案

```go
t.Run("behaves like a normal reader", func(t *testing.T) {
	rdr := NewCancellableReader(strings.NewReader("123456"))
	got := make([]byte, 3)
	_, err := rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "123")

	_, err = rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "456")
})
```

## 试着运行测试

```
./cancel_readers_test.go:12:10: undefined: NewCancellableReader
```
## 写最少的代码让测试能运行，并检查失败的测试输出

我们需要定义这个函数，它应该返回一个 `io.Reader`

```go
func NewCancellableReader(rdr io.Reader) io.Reader {
	return nil
}
```

如果你尝试运行它

```
=== RUN   TestCancelReaders
=== RUN   TestCancelReaders/behaves_like_a_normal_reader
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x10f8fb5]
```

如预期。

## 写足够的代码让它通过

目前，我们就直接返回传入的 `io.Reader`

```go
func NewCancellableReader(rdr io.Reader) io.Reader {
	return rdr
}
```

测试现在应该通过了。

我知道，我知道，这看起来傻又拘泥形式，但在冲进花哨工作之前，重要的是我们对自己没有破坏 `io.Reader` 的"正常"行为有 _某种_ 验证，这个测试会在我们前进时给我们信心。

## 先写测试

接下来我们需要尝试取消。

```go
t.Run("stops reading when cancelled", func(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	rdr := NewCancellableReader(ctx, strings.NewReader("123456"))
	got := make([]byte, 3)
	_, err := rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "123")

	cancel()

	n, err := rdr.Read(got)

	if err == nil {
		t.Error("expected an error after cancellation but didn't get one")
	}

	if n > 0 {
		t.Errorf("expected 0 bytes to be read after cancellation but %d were read", n)
	}
})
```

我们或多或少可以复制第一个测试，但现在我们：
- 创建一个带取消的 `context.Context`，这样可以在第一次读后 `cancel`
- 为了让我们的代码工作，我们需要把 `ctx` 传给我们的函数
- 然后我们断言在 `cancel` 之后什么都没读到

## 试着运行测试

```
./cancel_readers_test.go:33:30: too many arguments in call to NewCancellableReader
	have (context.Context, *strings.Reader)
	want (io.Reader)
```

## 写最少的代码让测试能运行，并检查失败的测试输出

编译器告诉我们要做什么；更新签名以接受一个 context

```go
func NewCancellableReader(ctx context.Context, rdr io.Reader) io.Reader {
	return rdr
}
```

（你也需要更新第一个测试，让它传入 `context.Background`）

你现在应该看到非常清晰的失败测试输出

```
=== RUN   TestCancelReaders
=== RUN   TestCancelReaders/stops_reading_when_cancelled
--- FAIL: TestCancelReaders (0.00s)
    --- FAIL: TestCancelReaders/stops_reading_when_cancelled (0.00s)
        cancel_readers_test.go:48: expected an error but didn't get one
        cancel_readers_test.go:52: expected 0 bytes to be read after cancellation but 3 were read
```

## 写足够的代码让它通过

到这里，从 Mat 和 David 的原帖里就是复制粘贴的过程，但我们仍然慢慢地、迭代地来。

我们知道我们需要一个类型，把我们要读取的 `io.Reader` 和 `context.Context` 封装起来，所以我们来创建它，并尝试从我们的函数中返回它，而不是返回原来的 `io.Reader`

```go
func NewCancellableReader(ctx context.Context, rdr io.Reader) io.Reader {
	return &readerCtx{
		ctx:      ctx,
		delegate: rdr,
	}
}

type readerCtx struct {
	ctx      context.Context
	delegate io.Reader
}
```

正如我在本书中多次强调的，慢慢来，让编译器帮你

```
./cancel_readers_test.go:60:3: cannot use &readerCtx literal (type *readerCtx) as type io.Reader in return argument:
	*readerCtx does not implement io.Reader (missing Read method)
```

抽象感觉对，但它没实现我们需要的接口（`io.Reader`），那我们加上方法。

```go
func (r *readerCtx) Read(p []byte) (n int, err error) {
	panic("implement me")
}
```

运行测试，它们应该可以 _编译_ 但会 panic。这仍是进展。

让我们通过 _委托_ 调用底层 `io.Reader` 来让第一个测试通过

```go
func (r readerCtx) Read(p []byte) (n int, err error) {
	return r.delegate.Read(p)
}
```

到这里我们的 happy path 测试又通过了，并且感觉我们的东西被很好地抽象了

为了让第二个测试通过，我们需要检查 `context.Context` 看它是否被取消了。

```go
func (r readerCtx) Read(p []byte) (n int, err error) {
	if err := r.ctx.Err(); err != nil {
		return 0, err
	}
	return r.delegate.Read(p)
}
```

所有测试现在应该都通过了。你会注意到我们如何返回来自 `context.Context` 的错误。这允许代码的调用方查看取消发生的各种原因，原文中对此有更多介绍。

## 总结

- 小接口很好，并且很容易组合
- 当你试图用一个东西增强另一个东西时（比如 `io.Reader`），你通常会想到[委托模式](https://en.wikipedia.org/wiki/Delegation_pattern)

> 在软件工程中，委托模式是一种面向对象的设计模式，它允许通过对象组合来达到与继承相同的代码复用。

- 开始这种工作的简单方法是包装你的委托对象，并写一个测试断言它行为与委托对象通常的行为一致，然后再开始组合其他部件以改变行为。这有助于你在朝目标编码时让事情保持正确工作
