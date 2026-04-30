# Time

[**本章的所有代码可以在这里找到**](https://github.com/quii/learn-go-with-tests/tree/main/time)

产品负责人希望我们扩展命令行应用的功能，帮助一群人玩德州扑克（Texas-Holdem Poker）。

## 关于扑克的最低限度信息

你不需要太了解扑克，只需要知道：在某些时间间隔之后，所有玩家都需要被通知一个不断增加的"盲注"（blind）值。

我们的应用会帮助跟踪盲注什么时候应该提高，以及提高到多少。

* 启动时它会询问有多少玩家在玩。这决定了"盲注"提高之前的时间长度。
  * 基础时间是 5 分钟。
  * 每多一名玩家，加 1 分钟。
  * 例如 6 名玩家，盲注间隔为 11 分钟。
* 盲注时间到了之后，游戏应当通知玩家新的盲注金额。
* 盲注从 100 筹码开始，然后是 200、400、600、1000、2000，并持续翻倍直到游戏结束（我们之前的"Ruth wins"功能仍然应当结束游戏）

## 代码回顾

在上一章里，我们开始了命令行应用的开发，它已经可以接受 `{name} wins` 的命令。下面是当前 `CLI` 代码的样子，但开始之前请确保你也熟悉了其他代码。

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
}

func NewCLI(store PlayerStore, in io.Reader) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
	}
}

func (cli *CLI) PlayPoker() {
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```

### `time.AfterFunc`

我们希望能够安排程序在不同的时间间隔（取决于玩家人数）打印盲注金额。

为了限制我们要做的事情的范围，我们暂时忽略玩家数量这一部分，假设有 5 名玩家，并测试 _每 10 分钟打印一次新的盲注金额_。

像往常一样，标准库已经为我们准备好了：[`func AfterFunc(d Duration, f func()) *Timer`](https://golang.org/pkg/time/#AfterFunc)

> `AfterFunc` waits for the duration to elapse and then calls f in its own goroutine. It returns a `Timer` that can be used to cancel the call using its Stop method.

### [`time.Duration`](https://golang.org/pkg/time/#Duration)

> A Duration represents the elapsed time between two instants as an int64 nanosecond count.

time 库提供了一些常量，让你把那些纳秒乘起来，对于我们将要做的场景来说更具可读性

```
5 * time.Second
```

当我们调用 `PlayPoker` 时，我们会安排所有的盲注提醒。

但测试这件事可能有点棘手。我们想要验证每个时间段都用正确的盲注金额安排了，但如果你看 `time.AfterFunc` 的签名，它的第二个参数是它要执行的函数。在 Go 中你不能比较函数，所以我们没办法测试传入的是什么函数。所以我们需要写一个 `time.AfterFunc` 的包装器，它会接收要执行的时间和要打印的金额，这样我们就可以监视它（spy on）。

## 先写测试

向我们的测试套件加一个新测试

```go
t.Run("it schedules printing of blind values", func(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(playerStore, in, blindAlerter)
	cli.PlayPoker()

	if len(blindAlerter.alerts) != 1 {
		t.Fatal("expected a blind alert to be scheduled")
	}
})
```

你会注意到我们做了一个 `SpyBlindAlerter`，我们尝试把它注入到 `CLI` 中，然后检查在调用 `PlayPoker` 之后是否安排了一个提醒。

（记住我们先做最简单的场景，然后再迭代。）

下面是 `SpyBlindAlerter` 的定义

```go
type SpyBlindAlerter struct {
	alerts []struct {
		scheduledAt time.Duration
		amount      int
	}
}

func (s *SpyBlindAlerter) ScheduleAlertAt(duration time.Duration, amount int) {
	s.alerts = append(s.alerts, struct {
		scheduledAt time.Duration
		amount      int
	}{duration, amount})
}
```

## 尝试运行测试

```
./CLI_test.go:32:27: too many arguments in call to poker.NewCLI
	have (*poker.StubPlayerStore, *strings.Reader, *SpyBlindAlerter)
	want (poker.PlayerStore, io.Reader)
```

## 写最少的代码让测试能跑起来，并检查失败的测试输出

我们加了一个新参数，编译器在抱怨。_严格来说_ 最少的代码量是让 `NewCLI` 接受一个 `*SpyBlindAlerter`，但我们稍微"作弊"一下，直接把这个依赖定义为一个接口。

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

然后把它加到构造函数里

```go
func NewCLI(store PlayerStore, in io.Reader, alerter BlindAlerter) *CLI
```

你的其他测试现在会失败，因为它们没有给 `NewCLI` 传 `BlindAlerter`。

监视 BlindAlerter 在其他测试里并不重要，所以在测试文件里加上

```go
var dummySpyAlerter = &SpyBlindAlerter{}
```

然后在其他测试里使用它来修复编译问题。把它命名为 "dummy" 让阅读测试的人很清楚它并不重要。

[> Dummy objects are passed around but never actually used. Usually they are just used to fill parameter lists.](https://martinfowler.com/articles/mocksArentStubs.html)

测试现在应该能编译，并且我们的新测试会失败。

```
=== RUN   TestCLI
=== RUN   TestCLI/it_schedules_printing_of_blind_values
--- FAIL: TestCLI (0.00s)
    --- FAIL: TestCLI/it_schedules_printing_of_blind_values (0.00s)
    	CLI_test.go:38: expected a blind alert to be scheduled
```

## 写足够的代码让它通过

我们需要把 `BlindAlerter` 作为字段加到 `CLI` 上，这样我们就可以在 `PlayPoker` 方法中引用它。

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
	alerter     BlindAlerter
}

func NewCLI(store PlayerStore, in io.Reader, alerter BlindAlerter) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
		alerter:     alerter,
	}
}
```

为了让测试通过，我们可以用任何参数调用 `BlindAlerter`

```go
func (cli *CLI) PlayPoker() {
	cli.alerter.ScheduleAlertAt(5*time.Second, 100)
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

接下来我们要检查它为 5 名玩家安排了我们期望的所有提醒

## 先写测试

```go
	t.Run("it schedules printing of blind values", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}
		blindAlerter := &SpyBlindAlerter{}

		cli := poker.NewCLI(playerStore, in, blindAlerter)
		cli.PlayPoker()

		cases := []struct {
			expectedScheduleTime time.Duration
			expectedAmount       int
		}{
			{0 * time.Second, 100},
			{10 * time.Minute, 200},
			{20 * time.Minute, 300},
			{30 * time.Minute, 400},
			{40 * time.Minute, 500},
			{50 * time.Minute, 600},
			{60 * time.Minute, 800},
			{70 * time.Minute, 1000},
			{80 * time.Minute, 2000},
			{90 * time.Minute, 4000},
			{100 * time.Minute, 8000},
		}

		for i, c := range cases {
			t.Run(fmt.Sprintf("%d scheduled for %v", c.expectedAmount, c.expectedScheduleTime), func(t *testing.T) {

				if len(blindAlerter.alerts) <= i {
					t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
				}

				alert := blindAlerter.alerts[i]

				amountGot := alert.amount
				if amountGot != c.expectedAmount {
					t.Errorf("got amount %d, want %d", amountGot, c.expectedAmount)
				}

				gotScheduledTime := alert.scheduledAt
				if gotScheduledTime != c.expectedScheduleTime {
					t.Errorf("got scheduled time of %v, want %v", gotScheduledTime, c.expectedScheduleTime)
				}
			})
		}
	})
```

表驱动测试在这里非常合适，能清晰地说明我们的需求是什么。我们遍历这个表，并通过 `SpyBlindAlerter` 检查提醒是否以正确的值被安排了。

## 尝试运行测试

你应该会看到很多失败，类似下面这样

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_schedules_printing_of_blind_values
    --- FAIL: TestCLI/it_schedules_printing_of_blind_values (0.00s)
=== RUN   TestCLI/it_schedules_printing_of_blind_values/100_scheduled_for_0s
        --- FAIL: TestCLI/it_schedules_printing_of_blind_values/100_scheduled_for_0s (0.00s)
        	CLI_test.go:71: got scheduled time of 5s, want 0s
=== RUN   TestCLI/it_schedules_printing_of_blind_values/200_scheduled_for_10m0s
        --- FAIL: TestCLI/it_schedules_printing_of_blind_values/200_scheduled_for_10m0s (0.00s)
        	CLI_test.go:59: alert 1 was not scheduled [{5000000000 100}]
```

## 写足够的代码让它通过

```go
func (cli *CLI) PlayPoker() {

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + 10*time.Minute
	}

	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

这并不比我们之前已有的复杂多少。我们现在只是遍历一个 `blinds` 数组，并以递增的 `blindTime` 调用调度器

## 重构

我们可以把已安排的提醒封装到一个方法里，让 `PlayPoker` 读起来更清晰。

```go
func (cli *CLI) PlayPoker() {
	cli.scheduleBlindAlerts()
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func (cli *CLI) scheduleBlindAlerts() {
	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + 10*time.Minute
	}
}
```

最后我们的测试看起来有点笨重。我们有两个匿名结构体表示同一件事：一个 `ScheduledAlert`。让我们把它重构成一个新类型，再做一些辅助函数来比较它们。

```go
type scheduledAlert struct {
	at     time.Duration
	amount int
}

func (s scheduledAlert) String() string {
	return fmt.Sprintf("%d chips at %v", s.amount, s.at)
}

type SpyBlindAlerter struct {
	alerts []scheduledAlert
}

func (s *SpyBlindAlerter) ScheduleAlertAt(at time.Duration, amount int) {
	s.alerts = append(s.alerts, scheduledAlert{at, amount})
}
```

我们给类型加了 `String()` 方法，这样在测试失败时它会打印得更漂亮

更新我们的测试以使用新类型

```go
t.Run("it schedules printing of blind values", func(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(playerStore, in, blindAlerter)
	cli.PlayPoker()

	cases := []scheduledAlert{
		{0 * time.Second, 100},
		{10 * time.Minute, 200},
		{20 * time.Minute, 300},
		{30 * time.Minute, 400},
		{40 * time.Minute, 500},
		{50 * time.Minute, 600},
		{60 * time.Minute, 800},
		{70 * time.Minute, 1000},
		{80 * time.Minute, 2000},
		{90 * time.Minute, 4000},
		{100 * time.Minute, 8000},
	}

	for i, want := range cases {
		t.Run(fmt.Sprint(want), func(t *testing.T) {

			if len(blindAlerter.alerts) <= i {
				t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
			}

			got := blindAlerter.alerts[i]
			assertScheduledAlert(t, got, want)
		})
	}
})
```

`assertScheduledAlert` 你自己来实现。

我们在这里花了不少时间写测试，并有点"调皮"地没有真正集成进我们的应用。在堆上更多需求之前，让我们先解决这一点。

试着运行 app，它会编译失败，抱怨 `NewCLI` 参数不够。

让我们创建一个 `BlindAlerter` 的实现，可以在我们的应用中使用。

新建 `blind_alerter.go`，把我们的 `BlindAlerter` 接口移过去，并添加下面的新内容

```go
package poker

import (
	"fmt"
	"os"
	"time"
)

type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

type BlindAlerterFunc func(duration time.Duration, amount int)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}

func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

记住，任何 _类型_ 都可以实现接口，不仅是 `struct`。如果你做的是一个对外暴露只有一个方法的接口的库，把同时暴露一个 `MyInterfaceFunc` 类型是常见的惯用做法。

这个类型会是一个 `func`，并且也实现你的接口。这样使用你接口的人就可以选择只用一个函数实现你的接口；而不必创建一个空的 `struct` 类型。

然后我们创建函数 `StdOutAlerter`，它的签名和这个函数类型一样，使用 `time.AfterFunc` 来调度它打印到 `os.Stdout`。

更新创建 `NewCLI` 的 `main`，看到效果

```go
poker.NewCLI(store, os.Stdin, poker.BlindAlerterFunc(poker.StdOutAlerter)).PlayPoker()
```

运行之前你可能想把 `CLI` 中 `blindTime` 的递增从 10 分钟改为 10 秒，这样你可以实际看到效果。

你应该看到它每 10 秒按预期打印盲注金额。注意你仍然可以在 CLI 中输入 `Shaun wins`，程序会按预期停止。

游戏不一定总是 5 个人玩，所以我们需要在游戏开始前提示用户输入玩家人数。

## 先写测试

为了检查我们提示了用户输入玩家数，我们需要记录写入 StdOut 的内容。我们之前做过几次了，我们知道 `os.Stdout` 是一个 `io.Writer`，所以如果我们用依赖注入的方式在测试里传入一个 `bytes.Buffer`，就可以检查我们的代码会写入什么。

我们暂时不关心这个测试中其他协作者，所以我们在测试文件里做了一些 dummy。

我们应该稍微警惕一下：现在 `CLI` 已经有 4 个依赖了，感觉它可能开始承担太多职责了。我们暂时先这样，看看在加这个新功能的过程中是否会冒出某种重构。

```go
var dummyBlindAlerter = &SpyBlindAlerter{}
var dummyPlayerStore = &poker.StubPlayerStore{}
var dummyStdIn = &bytes.Buffer{}
var dummyStdOut = &bytes.Buffer{}
```

下面是我们的新测试

```go
t.Run("it prompts the user to enter the number of players", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	cli := poker.NewCLI(dummyPlayerStore, dummyStdIn, stdout, dummyBlindAlerter)
	cli.PlayPoker()

	got := stdout.String()
	want := "Please enter the number of players: "

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

我们在 `main` 中传入会是 `os.Stdout` 的东西，并查看写入了什么。

## 尝试运行测试

```
./CLI_test.go:38:27: too many arguments in call to poker.NewCLI
	have (*poker.StubPlayerStore, *bytes.Buffer, *bytes.Buffer, *SpyBlindAlerter)
	want (poker.PlayerStore, io.Reader, poker.BlindAlerter)
```

## 写最少的代码让测试能跑起来，并检查失败的测试输出

我们有了一个新的依赖，所以要更新 `NewCLI`

```go
func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI
```

现在 _其他_ 测试会编译失败，因为它们没有给 `NewCLI` 传 `io.Writer`。

为其他测试加上 `dummyStdOut`。

新测试应该这样失败

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players
    --- FAIL: TestCLI/it_prompts_the_user_to_enter_the_number_of_players (0.00s)
    	CLI_test.go:46: got '', want 'Please enter the number of players: '
FAIL
```

## 写足够的代码让它通过

我们需要把新依赖添加到 `CLI` 中，以便在 `PlayPoker` 中引用

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
	out         io.Writer
	alerter     BlindAlerter
}

func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
		out:         out,
		alerter:     alerter,
	}
}
```

最后我们可以在游戏开始时写出提示

```go
func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, "Please enter the number of players: ")
	cli.scheduleBlindAlerts()
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

## 重构

我们的提示有重复的字符串，应该把它抽成一个常量

```go
const PlayerPrompt = "Please enter the number of players: "
```

在测试代码和 `CLI` 中都使用它。

现在我们需要传入一个数字并提取出来。我们能知道它是否达到了预期效果的唯一方法，是看安排了什么样的盲注提醒。

## 先写测试

```go
t.Run("it prompts the user to enter the number of players", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	in := strings.NewReader("7\n")
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(dummyPlayerStore, in, stdout, blindAlerter)
	cli.PlayPoker()

	got := stdout.String()
	want := poker.PlayerPrompt

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}

	cases := []scheduledAlert{
		{0 * time.Second, 100},
		{12 * time.Minute, 200},
		{24 * time.Minute, 300},
		{36 * time.Minute, 400},
	}

	for i, want := range cases {
		t.Run(fmt.Sprint(want), func(t *testing.T) {

			if len(blindAlerter.alerts) <= i {
				t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
			}

			got := blindAlerter.alerts[i]
			assertScheduledAlert(t, got, want)
		})
	}
})
```

哎呦！变化好多。

* 我们移除了 StdIn 的 dummy，改为传入一个 mock 版本，表示用户输入了 7
* 我们也移除了 blind alerter 上的 dummy，这样就能看到玩家数对调度产生了影响
* 我们测试了哪些提醒被安排了

## 尝试运行测试

测试应该能编译但失败，报告调度的时间不对，因为我们之前把游戏硬编码为基于 5 名玩家

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players
    --- FAIL: TestCLI/it_prompts_the_user_to_enter_the_number_of_players (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players/100_chips_at_0s
        --- PASS: TestCLI/it_prompts_the_user_to_enter_the_number_of_players/100_chips_at_0s (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players/200_chips_at_12m0s
```

## 写足够的代码让它通过

记住，我们可以为了让它工作做任何"出格"的事。一旦有了能工作的软件，我们就可以重构我们即将搞乱的部分！

```go
func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, PlayerPrompt)

	numberOfPlayers, _ := strconv.Atoi(cli.readLine())

	cli.scheduleBlindAlerts(numberOfPlayers)

	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func (cli *CLI) scheduleBlindAlerts(numberOfPlayers int) {
	blindIncrement := time.Duration(5+numberOfPlayers) * time.Minute

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + blindIncrement
	}
}
```

* 我们把 `numberOfPlayersInput` 读成一个字符串
* 我们用 `cli.readLine()` 获取用户输入，然后调用 `Atoi` 把它转成整数——忽略所有错误场景。我们之后需要为这个场景写测试。
* 这里我们改 `scheduleBlindAlerts`，让它接受玩家数。然后我们计算一个 `blindIncrement` 时间，遍历盲注金额时把它加到 `blindTime` 上

虽然新测试通过了，但很多其他测试失败了，因为我们的系统现在只有在用户输入数字开始游戏时才能工作。你需要修复这些测试，把用户输入改成数字加换行（这进一步暴露了我们当前方案的更多问题）。

## 重构

这一切感觉有点糟糕，对吧？让我们 **听听我们的测试**。

* 为了测试我们安排了一些提醒，我们设置了 4 个不同的依赖。当一个 _事物_ 在你的系统中需要很多依赖时，意味着它做的事情太多了。从视觉上我们也可以看到，测试看起来很乱。
* 在我看来，**我们需要在读取用户输入和我们想做的业务逻辑之间做一个更干净的抽象**
* 一个更好的测试是：_给定这样的用户输入，我们是否用正确的玩家数调用了一个新类型 `Game`_。
* 然后我们就把调度的测试提取到这个新 `Game` 的测试里。

我们可以先朝着 `Game` 重构，并且我们的测试应当继续通过。一旦我们做了想做的结构性变更，我们就可以考虑如何重构测试，让它反映我们新的关注点分离

记住，进行重构改动时，尽量让它们尽可能小，并不断重新运行测试。

先自己尝试一下。想想 `Game` 应该提供什么样的边界，以及 `CLI` 应该做什么。

现在 **不要** 改变 `NewCLI` 的对外接口，因为我们不希望同时改测试代码和客户端代码，那样要应付的太多了，可能把事情搞坏。

这是我得出的方案：

```go
// game.go
type Game struct {
	alerter BlindAlerter
	store   PlayerStore
}

func (p *Game) Start(numberOfPlayers int) {
	blindIncrement := time.Duration(5+numberOfPlayers) * time.Minute

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		p.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + blindIncrement
	}
}

func (p *Game) Finish(winner string) {
	p.store.RecordWin(winner)
}

// cli.go
type CLI struct {
	in   *bufio.Scanner
	out  io.Writer
	game *Game
}

func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI {
	return &CLI{
		in:  bufio.NewScanner(in),
		out: out,
		game: &Game{
			alerter: alerter,
			store:   store,
		},
	}
}

const PlayerPrompt = "Please enter the number of players: "

func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, PlayerPrompt)

	numberOfPlayersInput := cli.readLine()
	numberOfPlayers, _ := strconv.Atoi(strings.Trim(numberOfPlayersInput, "\n"))

	cli.game.Start(numberOfPlayers)

	winnerInput := cli.readLine()
	winner := extractWinner(winnerInput)

	cli.game.Finish(winner)
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins\n", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```

从"领域"角度看：

* 我们想 `Start` 一个 `Game`，指明有多少人在玩
* 我们想 `Finish` 一个 `Game`，宣布获胜者

新的 `Game` 类型为我们封装了这些。

通过这个改动，我们把 `BlindAlerter` 和 `PlayerStore` 传给了 `Game`，因为它现在负责提醒和存储结果。

我们的 `CLI` 现在只关心：

* 用现有依赖构造 `Game`（接下来我们会重构）
* 把用户输入解释为对 `Game` 的方法调用

我们要尽量避免做"大型"的重构，因为那会让我们处于测试失败的状态较长时间，犯错的机会增加。（如果你在大型/分布式团队工作，这格外重要）

我们要做的第一件事是重构 `Game`，让它注入到 `CLI` 中。我们会在测试中做最小的改动来配合这一点，然后看看怎么把测试拆分到解析用户输入和游戏管理这两个主题里。

我们现在要做的就是改 `NewCLI`

```go
func NewCLI(in io.Reader, out io.Writer, game *Game) *CLI {
	return &CLI{
		in:   bufio.NewScanner(in),
		out:  out,
		game: game,
	}
}
```

这已经感觉是个改进了。我们的依赖更少了，并且 _依赖列表反映了我们整体的设计目标_：CLI 关心输入/输出，把和游戏相关的动作委托给 `Game`。

如果你尝试编译会有问题，你应该可以自己修这些问题。先不要为 `Game` 做任何 mock，只需初始化 _真实的_ `Game` 来让一切编译并且测试通过。

为此你需要写一个构造函数

```go
func NewGame(alerter BlindAlerter, store PlayerStore) *Game {
	return &Game{
		alerter: alerter,
		store:   store,
	}
}
```

下面是修复某个测试设置的例子

```go
stdout := &bytes.Buffer{}
in := strings.NewReader("7\n")
blindAlerter := &SpyBlindAlerter{}
game := poker.NewGame(blindAlerter, dummyPlayerStore)

cli := poker.NewCLI(in, stdout, game)
cli.PlayPoker()
```

修复测试并回到绿色应该不会花太多力气（这就是重点！），但确保在下一阶段之前也修好 `main.go`。

```go
// main.go
game := poker.NewGame(poker.BlindAlerterFunc(poker.StdOutAlerter), store)
cli := poker.NewCLI(os.Stdin, os.Stdout, game)
cli.PlayPoker()
```

既然已经把 `Game` 提取出来，我们应当把游戏特有的断言移到独立于 CLI 的测试里。

这只是把我们的 `CLI` 测试复制一份，但依赖更少

```go
func TestGame_Start(t *testing.T) {
	t.Run("schedules alerts on game start for 5 players", func(t *testing.T) {
		blindAlerter := &poker.SpyBlindAlerter{}
		game := poker.NewGame(blindAlerter, dummyPlayerStore)

		game.Start(5)

		cases := []poker.ScheduledAlert{
			{At: 0 * time.Second, Amount: 100},
			{At: 10 * time.Minute, Amount: 200},
			{At: 20 * time.Minute, Amount: 300},
			{At: 30 * time.Minute, Amount: 400},
			{At: 40 * time.Minute, Amount: 500},
			{At: 50 * time.Minute, Amount: 600},
			{At: 60 * time.Minute, Amount: 800},
			{At: 70 * time.Minute, Amount: 1000},
			{At: 80 * time.Minute, Amount: 2000},
			{At: 90 * time.Minute, Amount: 4000},
			{At: 100 * time.Minute, Amount: 8000},
		}

		checkSchedulingCases(cases, t, blindAlerter)
	})

	t.Run("schedules alerts on game start for 7 players", func(t *testing.T) {
		blindAlerter := &poker.SpyBlindAlerter{}
		game := poker.NewGame(blindAlerter, dummyPlayerStore)

		game.Start(7)

		cases := []poker.ScheduledAlert{
			{At: 0 * time.Second, Amount: 100},
			{At: 12 * time.Minute, Amount: 200},
			{At: 24 * time.Minute, Amount: 300},
			{At: 36 * time.Minute, Amount: 400},
		}

		checkSchedulingCases(cases, t, blindAlerter)
	})

}

func TestGame_Finish(t *testing.T) {
	store := &poker.StubPlayerStore{}
	game := poker.NewGame(dummyBlindAlerter, store)
	winner := "Ruth"

	game.Finish(winner)
	poker.AssertPlayerWin(t, store, winner)
}
```

扑克游戏开始时发生什么的意图现在清晰多了。

确保也把游戏结束时的测试移过来。

一旦我们对游戏逻辑相关的测试已经移过来感到满意，就可以简化 CLI 测试，让它们更清楚地反映我们的预期职责

* 处理用户输入并在合适时调用 `Game` 的方法
* 发送输出
* 关键的是它不知道游戏的实际工作机制

为此，我们需要让 `CLI` 不再依赖具体的 `Game` 类型，而是接受一个带有 `Start(numberOfPlayers)` 和 `Finish(winner)` 的接口。然后我们就可以创建该类型的 spy，验证调用是否正确。

到这里我们意识到命名有时挺别扭。把 `Game` 重命名为 `TexasHoldem`（因为这是我们玩的游戏 _种类_），新的接口叫做 `Game`。这忠实于这样一种理念：我们的 CLI 并不知道实际玩的是什么游戏，也不知道 `Start` 和 `Finish` 时发生了什么。

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

把 `CLI` 内的所有 `*Game` 引用替换成 `Game`（我们的新接口）。一如既往，不断重新运行测试以确保我们重构期间一切都是绿色的。

现在我们已经把 `CLI` 与 `TexasHoldem` 解耦了，我们可以使用 spy 来检查我们期望调用 `Start` 和 `Finish` 时它们被以正确的参数调用。

创建一个实现 `Game` 的 spy

```go
type GameSpy struct {
	StartedWith  int
	FinishedWith string
}

func (g *GameSpy) Start(numberOfPlayers int) {
	g.StartedWith = numberOfPlayers
}

func (g *GameSpy) Finish(winner string) {
	g.FinishedWith = winner
}
```

把任何测试游戏特定逻辑的 `CLI` 测试，替换成对 `GameSpy` 调用情况的检查。这样我们的测试就清晰地反映了 CLI 的职责。

下面是修复其中一个测试的例子；剩下的自己试试，卡住了再看源码。

```go
	t.Run("it prompts the user to enter the number of players and starts the game", func(t *testing.T) {
		stdout := &bytes.Buffer{}
		in := strings.NewReader("7\n")
		game := &GameSpy{}

		cli := poker.NewCLI(in, stdout, game)
		cli.PlayPoker()

		gotPrompt := stdout.String()
		wantPrompt := poker.PlayerPrompt

		if gotPrompt != wantPrompt {
			t.Errorf("got %q, want %q", gotPrompt, wantPrompt)
		}

		if game.StartedWith != 7 {
			t.Errorf("wanted Start called with 7 but got %d", game.StartedWith)
		}
	})
```

既然有了清晰的关注点分离，检查 `CLI` 中关于 IO 的边界情况就更容易了。

我们需要处理用户在被提示输入玩家数时输入非数字值的场景：

我们的代码不应当开始游戏，应当向用户打印一条有用的错误信息然后退出。

## 先写测试

我们先确保游戏不会开始

```go
t.Run("it prints an error when a non numeric value is entered and does not start the game", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	in := strings.NewReader("Pies\n")
	game := &GameSpy{}

	cli := poker.NewCLI(in, stdout, game)
	cli.PlayPoker()

	if game.StartCalled {
		t.Errorf("game should not have started")
	}
})
```

你需要给 `GameSpy` 加一个字段 `StartCalled`，只在 `Start` 被调用时设置它

## 尝试运行测试

```
=== RUN   TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game
    --- FAIL: TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game (0.00s)
        CLI_test.go:62: game should not have started
```

## 写足够的代码让它通过

在我们调用 `Atoi` 的附近，只需要检查错误

```go
numberOfPlayers, err := strconv.Atoi(cli.readLine())

if err != nil {
	return
}
```

接下来我们需要告诉用户他们做错了什么，所以我们对打印到 `stdout` 的内容做断言。

## 先写测试

我们之前对打印到 `stdout` 的内容做过断言，所以可以暂时复制那段代码

```go
gotPrompt := stdout.String()

wantPrompt := poker.PlayerPrompt + "you're so silly"

if gotPrompt != wantPrompt {
	t.Errorf("got %q, want %q", gotPrompt, wantPrompt)
}
```

我们存的是写到 stdout 的 _所有_ 内容，所以仍然期望 `poker.PlayerPrompt`。然后我们再检查多打印了一段额外的内容。我们暂时不太在意确切的措辞，等重构时再处理。

## 尝试运行测试

```
=== RUN   TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game
    --- FAIL: TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game (0.00s)
        CLI_test.go:70: got 'Please enter the number of players: ', want 'Please enter the number of players: you're so silly'
```

## 写足够的代码让它通过

修改错误处理代码

```go
if err != nil {
	fmt.Fprint(cli.out, "you're so silly")
	return
}
```

## 重构

现在把这条信息重构为一个常量，就像 `PlayerPrompt`

```go
wantPrompt := poker.PlayerPrompt + poker.BadPlayerInputErrMsg
```

并放入更合适的信息

```go
const BadPlayerInputErrMsg = "Bad value received for number of players, please try again with a number"
```

最后我们关于发送到 `stdout` 内容的测试有点啰嗦，我们写一个断言函数来清理它。

```go
func assertMessagesSentToUser(t testing.TB, stdout *bytes.Buffer, messages ...string) {
	t.Helper()
	want := strings.Join(messages, "")
	got := stdout.String()
	if got != want {
		t.Errorf("got %q sent to stdout but expected %+v", got, messages)
	}
}
```

使用变长参数语法（`...string`）在这里很方便，因为我们需要对不定数量的消息做断言。

在两个对发送给用户的消息做断言的测试中都使用这个辅助函数。

还有一些测试可以借助一些 `assertX` 函数来改善，所以练习一下你的重构能力，把测试整理得读起来更舒服。

花点时间想一下我们驱动出的某些测试的价值。记住我们不想要超出必要的测试，你能 _在仍然对一切正常运行有信心的前提下_ 重构/移除其中一些吗？

下面是我得出的结果

```go
func TestCLI(t *testing.T) {

	t.Run("start game with 3 players and finish game with 'Chris' as winner", func(t *testing.T) {
		game := &GameSpy{}
		stdout := &bytes.Buffer{}

		in := userSends("3", "Chris wins")
		cli := poker.NewCLI(in, stdout, game)

		cli.PlayPoker()

		assertMessagesSentToUser(t, stdout, poker.PlayerPrompt)
		assertGameStartedWith(t, game, 3)
		assertFinishCalledWith(t, game, "Chris")
	})

	t.Run("start game with 8 players and record 'Cleo' as winner", func(t *testing.T) {
		game := &GameSpy{}

		in := userSends("8", "Cleo wins")
		cli := poker.NewCLI(in, dummyStdOut, game)

		cli.PlayPoker()

		assertGameStartedWith(t, game, 8)
		assertFinishCalledWith(t, game, "Cleo")
	})

	t.Run("it prints an error when a non numeric value is entered and does not start the game", func(t *testing.T) {
		game := &GameSpy{}

		stdout := &bytes.Buffer{}
		in := userSends("pies")

		cli := poker.NewCLI(in, stdout, game)
		cli.PlayPoker()

		assertGameNotStarted(t, game)
		assertMessagesSentToUser(t, stdout, poker.PlayerPrompt, poker.BadPlayerInputErrMsg)
	})
}
```

测试现在反映了 CLI 的主要能力：能从用户输入里读取多少人在玩、谁赢了，并能处理玩家数被输入了不正常值的情况。这样做对读者来说，`CLI` 做了什么、不做什么都很清晰。

如果用户没输入 `Ruth wins`，而是输入 `Lloyd is a killer` 会怎样？

通过为这个场景写测试并让它通过来完成本章。

## 总结

### 项目快速回顾

在过去的 5 章里，我们慢慢地通过 TDD 写了相当一部分代码

* 我们有两个应用，一个命令行应用和一个 Web 服务器。
* 这两个应用都依赖 `PlayerStore` 来记录获胜者
* Web 服务器还可以展示一个谁赢得最多的联赛积分榜
* 命令行应用通过追踪当前盲注值来帮助玩家玩扑克游戏。

### time.Afterfunc

一种在指定时长之后调度函数调用的非常方便的方式。非常值得花时间[查看 `time` 的文档](https://golang.org/pkg/time/)，因为它有许多帮你节省时间的函数和方法可用。

我最喜欢的一些是

* `time.After(duration)` 在时长到了之后返回一个 `chan Time`。所以如果你想在某个时间 _之后_ 做点什么，这能帮上忙。
* `time.NewTicker(duration)` 返回一个 `Ticker`，与上面类似，它返回一个 channel，但这个会每隔一段时间"滴答"一下，而不是只发一次。如果你想每隔 `N duration` 执行一些代码，这非常方便。

### 关注点分离的更多好例子

_一般来说_，把处理用户输入与响应的职责和领域代码分开是好习惯。在我们的命令行应用和 Web 服务器中你都能看到这一点。

我们的测试变乱了。我们有太多断言（检查这个输入、安排了这些提醒，等等）和太多依赖。我们直观地能看到它很乱；**听你的测试很重要**。

* 如果你的测试看起来乱，尝试重构它们。
* 如果你重构了仍然很乱，那很可能在指向你设计中的缺陷
* 这是测试真正的强项之一。

虽然测试和生产代码都有点乱，但有测试做后盾我们可以自由重构。

记住在这种情况下要始终走小步，并在每次改动后重新运行测试。

同时重构测试代码 _和_ 生产代码会很危险，所以我们先重构生产代码（在当前状态我们没法把测试改善多少），且不改它的接口，这样就可以在改动期间尽可能依赖我们的测试。_然后_ 在设计改善之后再重构测试。

重构之后，依赖列表反映了我们的设计目标。这是 DI 的另一个好处：它经常能起到记录意图的作用。当你依赖全局变量时，职责会变得非常不清晰。

## 一个函数实现接口的例子

当你定义一个只有一个方法的接口时，你可能想考虑定义一个 `MyInterfaceFunc` 类型作为补充，这样使用者可以仅用一个函数就实现你的接口。

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

// BlindAlerterFunc 让你可以用一个函数实现 BlindAlerter
type BlindAlerterFunc func(duration time.Duration, amount int)

// ScheduleAlertAt 是 BlindAlerterFunc 对 BlindAlerter 的实现
func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}
```

通过这样做，使用你的库的人就可以仅用一个函数实现你的接口。他们可以用[类型转换](https://go.dev/tour/basics/13) 把他们的函数转换成 `BlindAlerterFunc`，然后把它当 BlindAlerter 用（因为 `BlindAlerterFunc` 实现了 `BlindAlerter`）。

```go
game := poker.NewTexasHoldem(poker.BlindAlerterFunc(poker.StdOutAlerter), store)
```

这里更广泛的要点是：在 Go 中你可以给 _类型_ 添加方法，不仅是结构体。这是一个非常强大的特性，你可以用它以更方便的方式实现接口。

考虑到你不仅可以定义函数类型，还可以围绕其他类型定义类型，这样你就可以给它们加方法。

```go
type Blog map[string]string

func (b Blog) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, b[r.URL.Path])
}
```

这里我们创建了一个 HTTP handler，实现了一个非常简单的"博客"，它会用 URL 路径作为 key，从 map 中读取存储的文章。
