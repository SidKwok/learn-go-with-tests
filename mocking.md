# Mocking

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/mocking)**

你被要求写一个程序，从 3 开始倒数，每个数字打印在新的一行（每次间隔 1 秒），当数到零时打印 "Go!" 并退出。

```
3
2
1
Go!
```

我们将通过编写一个 `Countdown` 函数来解决这个问题，然后把它放进一个 `main` 程序里，看起来像这样：

```go
package main

func main() {
	Countdown()
}
```

虽然这是一个相当简单的程序，但要全面地测试它，我们仍然需要一如既往地采用 _迭代_ 和 _测试驱动_ 的方法。

我所说的迭代是什么意思？我们要确保每次都迈出最小的一步，让我们拥有 _可用的软件_。

我们不希望花很长时间写一份理论上经过一些拼凑后能工作的代码，因为这通常是开发者掉进兔子洞的方式。**能把需求切分成尽可能小的片段，从而拥有 _可工作的软件_，是一项重要的技能。**

我们可以这样划分工作并迭代：

- 打印 3
- 打印 3、2、1 和 Go!
- 每行之间等待一秒

## 先写测试

我们的软件需要打印到 stdout，我们在依赖注入那一节里看到了如何使用依赖注入 (DI) 来方便测试这一点。

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}

	Countdown(buffer)

	got := buffer.String()
	want := "3"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

如果你对 `buffer` 这类东西不熟悉，请重新阅读 [上一节](dependency-injection.md)。

我们知道我们想要 `Countdown` 函数把数据写到某个地方，而 `io.Writer` 是 Go 中以接口形式捕获这一点的事实标准方式。

- 在 `main` 中我们会发送到 `os.Stdout`，这样用户就能在终端看到打印出的倒计时。
- 在测试中我们会发送到 `bytes.Buffer`，这样我们的测试就能捕获生成了哪些数据。

## 尝试运行测试

`./countdown_test.go:11:2: undefined: Countdown`

## 写最少的代码让测试能运行，并检查失败的测试输出

定义 `Countdown`

```go
func Countdown() {}
```

再试一次

```
./countdown_test.go:11:11: too many arguments in call to Countdown
    have (*bytes.Buffer)
    want ()
```

编译器在告诉你函数签名应该是什么样的，所以更新它。

```go
func Countdown(out *bytes.Buffer) {}
```

`countdown_test.go:17: got '' want '3'`

完美！

## 写足够的代码让测试通过

```go
func Countdown(out *bytes.Buffer) {
	fmt.Fprint(out, "3")
}
```

我们使用 `fmt.Fprint`，它接受一个 `io.Writer`（比如 `*bytes.Buffer`）并向它发送一个 `string`。测试应该可以通过了。

## 重构

我们知道，虽然 `*bytes.Buffer` 能用，但用一个更通用的接口会更好。

```go
func Countdown(out io.Writer) {
	fmt.Fprint(out, "3")
}
```

重新运行测试，它们应该能通过。

为了完成这件事，我们现在把函数接到 `main` 里，这样我们就有了可工作的软件，可以让自己确信我们正在取得进展。

```go
package main

import (
	"fmt"
	"io"
	"os"
)

func Countdown(out io.Writer) {
	fmt.Fprint(out, "3")
}

func main() {
	Countdown(os.Stdout)
}
```

试着运行程序，为你的成果惊叹一下吧。

是的这看起来微不足道，但这种方法是我推荐用于任何项目的。**取一个薄薄的功能切片，让它端到端地工作，并由测试支撑。**

接下来我们可以让它打印 2、1 然后 "Go!"。

## 先写测试

通过先把整体的管道架好，我们就能安全且轻松地迭代我们的解决方案。我们不再需要停下来重新运行程序去确信它在工作，因为所有的逻辑都被测试了。

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}

	Countdown(buffer)

	got := buffer.String()
	want := `3
2
1
Go!`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

反引号语法是创建 `string` 的另一种方式，它可以让你包含像换行符这样的内容，对我们的测试来说很完美。

## 尝试运行测试

```
countdown_test.go:21: got '3' want '3
        2
        1
        Go!'
```
## 写足够的代码让测试通过

```go
func Countdown(out io.Writer) {
	for i := 3; i > 0; i-- {
		fmt.Fprintln(out, i)
	}
	fmt.Fprint(out, "Go!")
}
```

用 `for` 循环，通过 `i--` 倒数，并用 `fmt.Fprintln` 把数字加上换行符打印到 `out`。最后用 `fmt.Fprint` 在后面发送 "Go!"。

## 重构

除了把一些"魔法"值重构成命名常量之外，没什么可重构的了。

```go
const finalWord = "Go!"
const countdownStart = 3

func Countdown(out io.Writer) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
	}
	fmt.Fprint(out, finalWord)
}
```

如果你现在运行程序，你应该能得到期望的输出，但我们还没让它有戏剧性的、每秒一次的倒计时停顿。

Go 让你可以用 `time.Sleep` 实现这一点。试着把它加进我们的代码里。

```go
func Countdown(out io.Writer) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		time.Sleep(1 * time.Second)
	}

	fmt.Fprint(out, finalWord)
}
```

如果你运行程序，它会按我们想要的方式工作。

## Mocking

测试仍然能通过，软件也按预期工作，但我们有一些问题：
- 我们的测试要花 3 秒才能跑完。
    - 每一篇关于软件开发的有前瞻性的文章都强调了快速反馈循环的重要性。
    - **慢的测试会毁掉开发者的生产力**。
    - 想象一下如果需求变得更复杂，需要更多测试。我们能接受 `Countdown` 的每一个新测试都给测试运行多加 3 秒吗？
- 我们还没有测试函数的一个重要属性。

我们对 `Sleep`（睡眠）有一个依赖，需要把它抽出来，这样我们才能在测试中控制它。

如果我们能 _mock_ `time.Sleep`，我们就可以使用 _依赖注入_ 用它来代替"真实"的 `time.Sleep`，然后我们可以**监视它的调用**来对它们做断言。

## 先写测试

我们把依赖定义为接口。这样我们就可以在 `main` 里使用一个 _真实_ 的 Sleeper，在测试里使用一个 _spy sleeper_。通过使用接口，我们的 `Countdown` 函数对此一无所知，并为调用者增加了一些灵活性。

```go
type Sleeper interface {
	Sleep()
}
```

我做了一个设计决定：我们的 `Countdown` 函数不负责睡多长时间。这至少现在简化了我们的代码，并意味着函数的使用者可以按自己喜欢的方式配置睡眠时间。

现在我们需要为它做一个 _mock_，供我们的测试使用。

```go
type SpySleeper struct {
	Calls int
}

func (s *SpySleeper) Sleep() {
	s.Calls++
}
```

_spy_ 是一种 _mock_，可以记录依赖是如何被使用的。它们可以记录传入的参数、被调用了多少次等等。在我们的例子里，我们记录 `Sleep()` 被调用了多少次，这样就能在测试中检查。

更新测试，注入对我们 spy 的依赖，并断言 sleep 被调用了 3 次。

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}
	spySleeper := &SpySleeper{}

	Countdown(buffer, spySleeper)

	got := buffer.String()
	want := `3
2
1
Go!`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}

	if spySleeper.Calls != 3 {
		t.Errorf("not enough calls to sleeper, want 3 got %d", spySleeper.Calls)
	}
}
```

## 尝试运行测试

```
too many arguments in call to Countdown
    have (*bytes.Buffer, *SpySleeper)
    want (io.Writer)
```

## 写最少的代码让测试能运行，并检查失败的测试输出

我们需要更新 `Countdown` 让它接受我们的 `Sleeper`

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		time.Sleep(1 * time.Second)
	}

	fmt.Fprint(out, finalWord)
}
```

如果你再试一次，你的 `main` 会因为同样的原因无法编译

```
./main.go:26:11: not enough arguments in call to Countdown
    have (*os.File)
    want (io.Writer, Sleeper)
```

我们来创建一个 _真实_ 的 sleeper，让它实现我们需要的接口

```go
type DefaultSleeper struct{}

func (d *DefaultSleeper) Sleep() {
	time.Sleep(1 * time.Second)
}
```

然后我们可以像这样在真实的应用程序中使用它

```go
func main() {
	sleeper := &DefaultSleeper{}
	Countdown(os.Stdout, sleeper)
}
```

## 写足够的代码让测试通过

测试现在能编译了但还不能通过，因为我们仍然在调用 `time.Sleep` 而不是注入的依赖。我们来修复它。

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		sleeper.Sleep()
	}

	fmt.Fprint(out, finalWord)
}
```

测试应该能通过了，并且不再需要 3 秒。

### 还有一些问题

还有另一个重要的属性我们没有测试。

`Countdown` 应该在每次下一次打印之前 sleep，例如：

- `Print N`
- `Sleep`
- `Print N-1`
- `Sleep`
- `Print Go!`
- 等等

我们最近的改动只断言它 sleep 了 3 次，但这些 sleep 可能是不按顺序发生的。

写测试时，如果你不确信测试给了你足够的信心，那就破坏它！（不过要确保你已经先把改动提交到版本控制了）。把代码改成下面这样

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		sleeper.Sleep()
	}

	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
	}

	fmt.Fprint(out, finalWord)
}
```

如果你运行测试，它们应该仍然能通过，尽管实现是错的。

我们再用 spy 写一个新测试来检查操作的顺序是正确的。

我们有两个不同的依赖，我们想把它们的所有操作都记录到一个列表里。所以我们要创建 _一个 spy 同时给两者用_。

```go
type SpyCountdownOperations struct {
	Calls []string
}

func (s *SpyCountdownOperations) Sleep() {
	s.Calls = append(s.Calls, sleep)
}

func (s *SpyCountdownOperations) Write(p []byte) (n int, err error) {
	s.Calls = append(s.Calls, write)
	return
}

const write = "write"
const sleep = "sleep"
```

我们的 `SpyCountdownOperations` 同时实现了 `io.Writer` 和 `Sleeper`，把每个调用都记录到一个切片里。在这个测试中我们只关心操作的顺序，所以仅把它们记录为命名操作的列表就足够了。

我们现在可以在测试套件里加一个子测试，验证 sleep 和 print 是按我们期望的顺序进行的

```go
t.Run("sleep before every print", func(t *testing.T) {
	spySleepPrinter := &SpyCountdownOperations{}
	Countdown(spySleepPrinter, spySleepPrinter)

	want := []string{
		write,
		sleep,
		write,
		sleep,
		write,
		sleep,
		write,
	}

	if !reflect.DeepEqual(want, spySleepPrinter.Calls) {
		t.Errorf("wanted calls %v got %v", want, spySleepPrinter.Calls)
	}
})
```

这个测试现在应该会失败。把 `Countdown` 还原回原来的样子来修复测试。

我们现在有两个测试在监视 `Sleeper`，所以可以重构测试，让一个测试用来测试打印的内容，另一个用来确保我们在打印之间睡眠。最后，我们可以删除第一个 spy，因为它已经不再被使用了。

```go
func TestCountdown(t *testing.T) {

	t.Run("prints 3 to Go!", func(t *testing.T) {
		buffer := &bytes.Buffer{}
		Countdown(buffer, &SpyCountdownOperations{})

		got := buffer.String()
		want := `3
2
1
Go!`

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})

	t.Run("sleep before every print", func(t *testing.T) {
		spySleepPrinter := &SpyCountdownOperations{}
		Countdown(spySleepPrinter, spySleepPrinter)

		want := []string{
			write,
			sleep,
			write,
			sleep,
			write,
			sleep,
			write,
		}

		if !reflect.DeepEqual(want, spySleepPrinter.Calls) {
			t.Errorf("wanted calls %v got %v", want, spySleepPrinter.Calls)
		}
	})
}
```

我们的函数和它的两个重要属性现在都被恰当地测试了。

## 扩展 Sleeper 使其可配置

让 `Sleeper` 可配置会是一个不错的特性。这意味着我们可以在主程序中调整 sleep 时长。

### 先写测试

我们先创建一个新类型 `ConfigurableSleeper`，它接受我们用于配置和测试所需的内容。

```go
type ConfigurableSleeper struct {
	duration time.Duration
	sleep    func(time.Duration)
}
```

我们用 `duration` 来配置睡眠时长，用 `sleep` 作为传入睡眠函数的方式。`sleep` 的签名与 `time.Sleep` 相同，这让我们可以在真实实现中使用 `time.Sleep`，并在测试中使用下面的 spy：

```go
type SpyTime struct {
	durationSlept time.Duration
}

func (s *SpyTime) SetDurationSlept(duration time.Duration) {
	s.durationSlept = duration
}
```

有了 spy，我们可以为可配置的 sleeper 创建一个新测试。

```go
func TestConfigurableSleeper(t *testing.T) {
	sleepTime := 5 * time.Second

	spyTime := &SpyTime{}
	sleeper := ConfigurableSleeper{sleepTime, spyTime.SetDurationSlept}
	sleeper.Sleep()

	if spyTime.durationSlept != sleepTime {
		t.Errorf("should have slept for %v but slept for %v", sleepTime, spyTime.durationSlept)
	}
}
```

这个测试应该没什么新东西，它的搭建和之前的 mock 测试非常相似。

### 尝试运行测试
```
sleeper.Sleep undefined (type ConfigurableSleeper has no field or method Sleep, but does have sleep)

```

你应该看到一条非常清晰的错误消息，表明我们没有在 `ConfigurableSleeper` 上创建 `Sleep` 方法。

### 写最少的代码让测试能运行，并检查失败的测试输出
```go
func (c *ConfigurableSleeper) Sleep() {
}
```

实现了新的 `Sleep` 函数后我们有了一个失败的测试。

```
countdown_test.go:56: should have slept for 5s but slept for 0s
```

### 写足够的代码让测试通过

我们现在需要做的就是为 `ConfigurableSleeper` 实现 `Sleep` 函数。

```go
func (c *ConfigurableSleeper) Sleep() {
	c.sleep(c.duration)
}
```

通过这个改动，所有测试应该再次通过，而你可能会想，主程序根本没变，为什么要这么折腾？希望在下面这一节看完后会清楚起来。

### 清理和重构

我们最后需要做的就是真的在 main 函数里使用 `ConfigurableSleeper`。

```go
func main() {
	sleeper := &ConfigurableSleeper{1 * time.Second, time.Sleep}
	Countdown(os.Stdout, sleeper)
}
```

如果我们运行测试并手动运行程序，可以看到所有行为都保持不变。

既然我们使用了 `ConfigurableSleeper`，现在可以安全地删除 `DefaultSleeper` 实现了。这样就把程序收尾了，我们有了一个更[通用](https://stackoverflow.com/questions/19291776/whats-the-difference-between-abstraction-and-generalization)的、可以任意倒数时长的 Sleeper。

## 但 mock 不是邪恶的吗？

你可能听说过 mock 是邪恶的。就像软件开发中的任何东西，它都可能被用作邪恶之事，就像 [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) 一样。

人们通常会陷入糟糕的境地，是因为他们不 _听他们的测试_，并且 _不尊重重构阶段_。

如果你的 mock 代码变得复杂，或者你为了测试某样东西必须 mock 大量东西，你应该 _倾听_ 那种糟糕的感觉，并思考你的代码。通常这是一个信号，表明：

- 你正在测试的东西做了太多事情（因为它有太多依赖需要 mock）
  - 把模块拆开，让它做更少的事
- 它的依赖太细粒度
  - 想想你怎么把这些依赖中的一些合并成一个有意义的模块
- 你的测试太关注实现细节
  - 优先测试预期的行为，而不是实现

通常大量的 mock 指向你代码中的 _糟糕抽象_。

**人们在这里看到的是 TDD 的弱点，但实际上这是它的优势**，糟糕的测试代码往往是糟糕设计的结果，或者更好听的说法是，设计良好的代码易于测试。

### 但 mock 和测试还是让我的生活很艰难！

遇到过这种情况吗？

- 你想做一些重构
- 为此你最终改了大量测试
- 你开始质疑 TDD，并在 Medium 上发表一篇题为"考虑 Mock 的危害"的帖子

这通常是你测试了太多 _实现细节_ 的标志。试着让你的测试测试 _有用的行为_，除非实现对系统的运行方式真的很重要。

有时很难知道究竟要在 _什么层级_ 测试，但这里是我尝试遵循的一些思考过程和规则：

- **重构的定义就是代码改变了但行为保持不变**。如果你决定做一些重构，理论上你应该能在不改任何测试的情况下提交。所以写测试的时候问问自己
  - 我是在测试我想要的行为，还是实现细节？
  - 如果我重构这段代码，我会需要对测试做大量改动吗？
- 虽然 Go 让你可以测试私有函数，但我会避免这样做，因为私有函数是支持公共行为的实现细节。测试公共行为。Sandi Metz 把私有函数描述为"较不稳定"，你不希望把测试与它们耦合在一起。
- 我觉得如果一个测试用了**超过 3 个 mock，那就是一个红色信号** —— 是时候重新考虑设计了
- 谨慎使用 spy。Spy 让你看到你正在编写的算法的内部，这可能非常有用，但这意味着你的测试代码与实现之间的耦合更紧。**如果你打算监视这些细节，要确保你真的关心它们**

#### 我不能直接用 mock 框架吗？

Mock 不需要任何魔法，相对简单；使用框架反而会让 mock 看起来比实际更复杂。我们在本章不使用自动 mock，是为了让我们：

- 更好地理解如何 mock
- 练习实现接口

在协作项目中，自动生成 mock 是有价值的。在团队里，mock 生成工具可以编码出测试替身的一致性。这能避免风格不一致的测试替身，从而避免风格不一致的测试。

你只应该使用根据接口生成测试替身的 mock 生成器。任何过度规定如何写测试，或使用大量"魔法"的工具，都可以扔到海里去。

## 总结

### 关于 TDD 方法的更多内容

- 当面对不那么平凡的例子时，把问题分解为"薄薄的纵向切片"。尽快达到 _有可工作软件并由测试支撑_ 的状态，避免掉进兔子洞或者采用"大爆炸"式的方法。
- 一旦你有了一些可工作的软件，应该就能更容易地 _以小步迭代_，直到你抵达你需要的软件。

> "什么时候用迭代式开发？你应该只在那些你想要成功的项目上使用迭代式开发。"

Martin Fowler。

### Mocking

- **没有 mock，你代码中重要的部分将无法被测试**。在我们的例子里，我们就无法测试我们的代码在每次打印之间停顿了，但还有无数其他例子。调用一个 _可能_ 失败的服务？想测试系统在某种特定状态下？没有 mock 很难测试这些场景。
- 没有 mock，你可能必须搭建数据库和其他第三方服务，仅仅是为了测试简单的业务规则。你的测试很可能会很慢，导致**慢反馈循环**。
- 因为必须启动数据库或 web 服务来测试某样东西，你的测试很可能会因为这些服务的不可靠性而变得**脆弱**。

一旦开发者了解了 mock，就很容易过度测试系统的每一个方面，就 _它如何工作_ 而非 _它做什么_ 而言。要时刻留意**你测试的价值**以及它们对未来重构的影响。

在这篇关于 mock 的文章中，我们只涉及了 **Spy**，它是一种 mock。Mock 是一种"测试替身"。

> [测试替身是一个通用术语，指任何用产品对象替换以进行测试的情况。](https://martinfowler.com/bliki/TestDouble.html)

在测试替身之下，有各种类型，比如 stub、spy 以及当然 mock！查看 [Martin Fowler 的文章](https://martinfowler.com/bliki/TestDouble.html) 了解更多细节。

## 附加内容 - Go 1.23 的迭代器示例

在 Go 1.23 中 [引入了迭代器](https://tip.golang.org/doc/go1.23)。我们可以以多种方式使用迭代器，在这种情况下，我们可以做一个 `countdownFrom` 迭代器，它将以倒序返回要倒数的数字。

在我们深入了解如何编写自定义迭代器之前，我们先看看怎么使用它们。比起写一个看起来相当命令式的循环来从一个数字倒数，我们可以让这段代码看起来更具表达力，方法是 `range` 遍历我们自定义的 `countdownFrom` 迭代器。

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := range countDownFrom(3) {
		fmt.Fprintln(out, i)
		sleeper.Sleep()
	}

	fmt.Fprint(out, finalWord)
}
```

要写一个像 `countDownFrom` 这样的迭代器，你需要以特定方式编写一个函数。摘自文档：

    "for-range" 循环中的 "range" 子句现在接受以下类型的迭代器函数
        func(func() bool)
        func(func(K) bool)
        func(func(K, V) bool)

（`K` 和 `V` 分别代表键和值的类型。）

在我们的例子里，我们没有键，只有值。Go 还提供了一个便利类型 `iter.Seq[T]`，它是 `func(func(T) bool)` 的类型别名。

```go
func countDownFrom(from int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := from; i > 0; i-- {
			if !yield(i) {
				return
			}
		}
	}
}
```

这是一个简单的迭代器，它会按倒序产出数字，从 `from` 开始 —— 完美适合我们的用例。
