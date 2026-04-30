# 命令行与项目结构

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/command-line)**

我们的产品负责人现在想要 _转向_，引入第二个应用 —— 一个命令行应用。

目前，它只需要在用户输入 `Ruth wins` 时记录一名玩家的胜利。最终的目的是成为一个帮助用户玩扑克的工具。

产品负责人希望两个应用共享数据库，这样根据新应用记录的胜利情况，联赛榜也会更新。

## 代码回顾

我们有一个带 `main.go` 文件的应用，它启动了一个 HTTP 服务器。HTTP 服务器对这次练习来说不太重要，但它使用的抽象很重要。它依赖一个 `PlayerStore`。

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() League
}
```

在上一章中，我们做了一个 `FileSystemPlayerStore` 实现了那个接口。我们应该可以在新应用中复用其中一些。

## 先做一些项目重构

我们的项目现在需要创建两个二进制文件，我们已有的 web 服务器和命令行应用。

在我们正式开始新工作之前，我们应该调整项目结构以适应这一点。

到目前为止所有代码都活在一个文件夹中，路径看起来像这样

`$GOPATH/src/github.com/your-name/my-app`

为了在 Go 中创建一个应用，你需要在 `package main` 包内有一个 `main` 函数。到目前为止我们所有的"领域"代码都活在 `package main` 中，我们的 `func main` 可以引用所有东西。

到目前为止这没问题，并且 _不要_ 在包结构上过度设计是个好习惯。如果你花时间浏览标准库，你会发现极少有大量文件夹和复杂结构的情况。

值得欣慰的是，_当你需要时_ 添加结构是相当直接的。

在已有项目内创建一个 `cmd` 目录，里面再创建一个 `webserver` 目录（例如 `mkdir -p cmd/webserver`）。

把 `main.go` 移到那里面。

如果你装了 `tree`，运行它，你的结构应该看起来像这样

```
.
|-- file_system_store.go
|-- file_system_store_test.go
|-- cmd
|   |-- webserver
|       |-- main.go
|-- league.go
|-- server.go
|-- server_integration_test.go
|-- server_test.go
|-- tape.go
|-- tape_test.go
```

我们现在实际上把应用代码和库代码分开了，但我们需要更改一些包名。记住，当你构建一个 Go 应用时，它的包 _必须_ 是 `main`。

把所有其他代码改为名为 `poker` 的包。

最后，我们需要把这个包导入到 `main.go`，这样我们就可以用它来创建 web 服务器。然后我们可以通过使用 `poker.FunctionName` 来使用我们的库代码。

路径在你的电脑上会不同，但应该类似这样：

```go
// cmd/webserver/main.go
package main

import (
	"github.com/quii/learn-go-with-tests/command-line/v1"
	"log"
	"net/http"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	server := poker.NewPlayerServer(store)

	log.Fatal(http.ListenAndServe(":5000", server))
}
```

完整的路径可能看起来有些刺眼，但这就是你如何把 _任何_ 公开可用的库导入到你的代码中。

通过把我们的领域代码分离到一个独立的包，并提交到像 GitHub 这样的公共仓库，任何 Go 开发者都可以写他们自己的代码，导入那个包，使用我们已经写好的特性。第一次运行时它会抱怨它不存在，但你只需运行 `go get`。

此外，用户可以查看 [pkg.go.dev 上的文档](https://pkg.go.dev/github.com/quii/learn-go-with-tests/command-line/v1)。

### 最后的检查

- 在根目录运行 `go test`，检查测试还能通过
- 进入我们的 `cmd/webserver` 目录运行 `go run main.go`
  - 访问 `http://localhost:5000/league`，你应该看到它仍在工作

### Walking skeleton

在我们正式开始写测试之前，我们先添加一个新应用让我们的项目可以构建。在 `cmd` 内再创建一个目录，叫 `cli`（command line interface），添加一个 `main.go`，内容如下

```go
// cmd/cli/main.go
package main

import "fmt"

func main() {
	fmt.Println("Let's play poker")
}
```

我们要解决的第一个需求是当用户输入 `{PlayerName} wins` 时记录一次胜利。

## 先写测试

我们知道我们需要做一个叫 `CLI` 的东西，让我们能够 `Play` 扑克。它需要读取用户输入，然后把胜利记录到一个 `PlayerStore`。

不过在我们想得太远之前，我们先写一个测试，检查它按我们想要的方式与 `PlayerStore` 集成。

在 `CLI_test.go` 中（在项目的根目录，不是在 `cmd` 内）

```go
// CLI_test.go
package poker

import "testing"

func TestCLI(t *testing.T) {
	playerStore := &StubPlayerStore{}
	cli := &CLI{playerStore}
	cli.PlayPoker()

	if len(playerStore.winCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}
}
```

- 我们可以用其他测试中的 `StubPlayerStore`
- 我们把依赖传入到尚不存在的 `CLI` 类型
- 通过未写出的 `PlayPoker` 方法触发游戏
- 检查是否记录了一次胜利

## 尝试运行测试

```
# github.com/quii/learn-go-with-tests/command-line/v2
./cli_test.go:25:10: undefined: CLI
```

## 写最少的代码让测试能运行，并检查失败的测试输出

到这里，你应该足够熟练，能创建我们的新 `CLI` 结构体（带有依赖对应的字段）并添加一个方法。

你最终的代码应该是这样

```go
// CLI.go
package poker

type CLI struct {
	playerStore PlayerStore
}

func (cli *CLI) PlayPoker() {}
```

记住，我们只是想让测试运行起来，这样可以检查测试按我们希望的方式失败

```
--- FAIL: TestCLI (0.00s)
    cli_test.go:30: expected a win call but didn't get any
FAIL
```

## 写足够的代码让测试通过

```go
//CLI.go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Cleo")
}
```

这样应该让测试通过了。

接下来，我们需要模拟从 `Stdin`（用户的输入）读取数据，这样我们可以为特定玩家记录胜利。

我们扩展测试来运行这个。

## 先写测试

```go
//CLI_test.go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &StubPlayerStore{}

	cli := &CLI{playerStore, in}
	cli.PlayPoker()

	if len(playerStore.winCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}

	got := playerStore.winCalls[0]
	want := "Chris"

	if got != want {
		t.Errorf("didn't record correct winner, got %q, want %q", got, want)
	}
}
```

`os.Stdin` 是我们将在 `main` 中用来捕获用户输入的东西。它底层是一个 `*File`，这意味着它实现了 `io.Reader`，到现在我们都知道这是捕获文本的便捷方式。

我们在测试中使用便利的 `strings.NewReader` 创建一个 `io.Reader`，把我们期望用户输入的内容填进去。

## 尝试运行测试

`./CLI_test.go:12:32: too many values in struct initializer`

## 写最少的代码让测试能运行，并检查失败的测试输出

我们需要把新的依赖加到 `CLI` 中。

```go
//CLI.go
type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}
```

```
--- FAIL: TestCLI (0.00s)
    CLI_test.go:23: didn't record the correct winner, got 'Cleo', want 'Chris'
FAIL
```

## 写足够的代码让测试通过

记得先做最简单的事情

```go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Chris")
}
```

测试通过了。我们接下来再加一个测试，迫使我们写一些真实的代码，但首先，让我们重构。

## 重构

在 `server_test` 中我们之前做过类似这里的检查，看胜利是否被记录。把那个断言 DRY 成一个辅助函数

```go
//server_test.go
func assertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != winner {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
	}
}
```

现在替换 `server_test.go` 和 `CLI_test.go` 中的断言。

测试现在应该读起来像这样

```go
//CLI_test.go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &StubPlayerStore{}

	cli := &CLI{playerStore, in}
	cli.PlayPoker()

	assertPlayerWin(t, playerStore, "Chris")
}
```

现在我们 _再_ 写一个测试，使用不同的用户输入来迫使我们真的去读它。

## 先写测试

```go
//CLI_test.go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &StubPlayerStore{}

		cli := &CLI{playerStore, in}
		cli.PlayPoker()

		assertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &StubPlayerStore{}

		cli := &CLI{playerStore, in}
		cli.PlayPoker()

		assertPlayerWin(t, playerStore, "Cleo")
	})

}
```

## 尝试运行测试

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/record_chris_win_from_user_input
    --- PASS: TestCLI/record_chris_win_from_user_input (0.00s)
=== RUN   TestCLI/record_cleo_win_from_user_input
    --- FAIL: TestCLI/record_cleo_win_from_user_input (0.00s)
        CLI_test.go:27: did not store correct winner got 'Chris' want 'Cleo'
FAIL
```

## 写足够的代码让测试通过

我们将使用 [`bufio.Scanner`](https://golang.org/pkg/bufio/) 从 `io.Reader` 读取输入。

> bufio 包实现了带缓冲的 I/O。它包装了一个 io.Reader 或 io.Writer 对象，创建另一个对象（Reader 或 Writer），这个对象同样实现了对应接口，但提供了缓冲以及对文本 I/O 的一些帮助。

把代码更新为下面这样

```go
//CLI.go
type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}

func (cli *CLI) PlayPoker() {
	reader := bufio.NewScanner(cli.in)
	reader.Scan()
	cli.playerStore.RecordWin(extractWinner(reader.Text()))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}
```

测试现在应该可以通过了。

- `Scanner.Scan()` 会读取直到换行符。
- 然后我们用 `Scanner.Text()` 返回 scanner 读到的 `string`。

既然我们有了一些通过的测试，我们应该把这接到 `main` 里。记住我们应该总是尽快做到拥有完全集成、可工作的软件。

在 `main.go` 中添加下面的内容并运行（你可能需要调整第二个依赖的路径以匹配你电脑上的）

```go
package main

import (
	"fmt"
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	fmt.Println("Let's play poker")
	fmt.Println("Type {Name} wins to record a win")

	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.CLI{store, os.Stdin}
	game.PlayPoker()
}
```

你应该会得到一个错误

```
command-line/v3/cmd/cli/main.go:32:25: implicit assignment of unexported field 'playerStore' in poker.CLI literal
command-line/v3/cmd/cli/main.go:32:34: implicit assignment of unexported field 'in' in poker.CLI literal
```

这里发生的事情是因为我们在试图给 `CLI` 中的 `playerStore` 和 `in` 字段赋值。这些是未导出（私有）的字段。我们 _可以_ 在测试代码中这样做，因为我们的测试和 `CLI` 在同一个包中（`poker`）。但我们的 `main` 在 `main` 包中，所以它没有访问权限。

这突显了 _集成你的工作_ 的重要性。我们正确地把 `CLI` 的依赖设为私有（因为我们不希望它们暴露给 `CLI` 的用户），但还没有为用户提供一种构造它的方法。

有没有办法更早地发现这个问题？

### `package mypackage_test`

到目前为止的所有其他例子里，当我们做一个测试文件时，我们都把它声明为与正在测试的包相同的包。

这没问题，并且意味着在偶尔我们想测试包内部的某些东西时，我们有访问未导出类型的权限。

但是鉴于我们 _一般_ 倡导 _不_ 测试内部的东西，Go 能帮助强制执行这一点吗？如果我们能像我们的 `main` 那样只能访问导出类型来测试我们的代码，会怎样？

当你写一个有多个包的项目时，我强烈建议你的测试包名以 `_test` 结尾。这样做之后你将只能访问你包内的公共类型。这对这个具体情况有帮助，但也有助于强制执行只测试公共 API 的纪律。如果你仍希望测试内部，可以单独做一个使用你想要测试的包的测试。

TDD 中有一句格言是，如果你不能测试你的代码，那对代码的用户来说集成它可能也很难。使用 `package foo_test` 会迫使你像导入它的用户那样测试你的代码，从而帮助解决这个问题。

在修复 `main` 之前，让我们把 `CLI_test.go` 内的测试包改为 `poker_test`。

如果你的 IDE 配置得不错，你会突然看到一大片红色！如果你运行编译器，你会得到下面的错误

```
./CLI_test.go:12:19: undefined: StubPlayerStore
./CLI_test.go:17:3: undefined: assertPlayerWin
./CLI_test.go:22:19: undefined: StubPlayerStore
./CLI_test.go:27:3: undefined: assertPlayerWin
```

我们现在又碰到了关于包设计的更多问题。为了测试我们的软件，我们做了未导出的 stub 和辅助函数，它们现在不再可供我们在 `CLI_test` 中使用，因为这些辅助函数定义在 `poker` 包的 `_test.go` 文件中。

#### 我们想让 stub 和辅助函数"公开"吗？

这是一个主观的讨论。有人可能会争辩说，你不希望用方便测试的代码污染你包的 API。

在 Mitchell Hashimoto 的演讲 ["Advanced Testing with Go"](https://speakerdeck.com/mitchellh/advanced-testing-with-go?slide=53) 中，描述了在 HashiCorp 他们如何倡导这样做，使得包的用户可以在写测试时不必重新发明轮子去写 stub。在我们的例子里，这意味着任何使用我们 `poker` 包的人，如果想跟我们的代码工作，都不必创建他们自己的 stub `PlayerStore`。

从轶事上讲，我在其他共享包中使用过这种技术，事实证明它在用户与我们的包集成时为他们节省了大量时间，非常有用。

所以我们创建一个名为 `testing.go` 的文件，把我们的 stub 和辅助函数加进去。

```go
// testing.go
package poker

import "testing"

type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}

func (s *StubPlayerStore) GetLeague() League {
	return s.league
}

func AssertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != winner {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
	}
}

// todo for you - the rest of the helpers
```

如果你希望这些辅助函数对你包的导入者开放，需要让它们公开（记住导出是通过开头大写字母完成的）。

在我们的 `CLI` 测试中，你需要像在不同的包中使用代码那样调用代码。

```go
//CLI_test.go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := &poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := &poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Cleo")
	})

}
```

你现在会看到我们遇到了和在 `main` 中相同的问题

```
./CLI_test.go:15:26: implicit assignment of unexported field 'playerStore' in poker.CLI literal
./CLI_test.go:15:39: implicit assignment of unexported field 'in' in poker.CLI literal
./CLI_test.go:25:26: implicit assignment of unexported field 'playerStore' in poker.CLI literal
./CLI_test.go:25:39: implicit assignment of unexported field 'in' in poker.CLI literal
```

绕过这个问题的最简单方法是像我们对其他类型那样做一个构造函数。我们也将更改 `CLI`，让它存储一个 `bufio.Scanner` 而不是 reader，因为它现在会在构造时自动包装。

```go
//CLI.go
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
```

通过这样做，我们可以简化和重构我们的读取代码

```go
//CLI.go
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

把测试改为使用构造函数，我们就应该回到测试通过的状态。

最后，我们可以回到新的 `main.go`，使用刚做好的构造函数

```go
//cmd/cli/main.go
game := poker.NewCLI(store, os.Stdin)
```

试着运行它，输入 "Bob wins"。

### 重构

在我们的两个应用中存在一些重复，都是打开一个文件并从其内容创建一个 `file_system_store`。这感觉像是我们包设计的一个小弱点，所以我们应该在包中做一个函数来封装从路径打开文件并返回 `PlayerStore` 的过程。

```go
//file_system_store.go
func FileSystemPlayerStoreFromFile(path string) (*FileSystemPlayerStore, func(), error) {
	db, err := os.OpenFile(path, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		return nil, nil, fmt.Errorf("problem opening %s %v", path, err)
	}

	closeFunc := func() {
		db.Close()
	}

	store, err := NewFileSystemPlayerStore(db)

	if err != nil {
		return nil, nil, fmt.Errorf("problem creating file system player store, %v ", err)
	}

	return store, closeFunc, nil
}
```

现在重构两个应用，使用这个函数来创建 store。

#### CLI 应用代码

```go
// cmd/cli/main.go
package main

import (
	"fmt"
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	fmt.Println("Let's play poker")
	fmt.Println("Type {Name} wins to record a win")
	poker.NewCLI(store, os.Stdin).PlayPoker()
}
```

#### Web 服务器应用代码

```go
// cmd/webserver/main.go
package main

import (
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"net/http"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	server := poker.NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

注意这种对称性：尽管是不同的用户接口，搭建几乎是一致的。这感觉像是对我们目前设计的良好验证。
还要注意 `FileSystemPlayerStoreFromFile` 返回了一个关闭函数，所以一旦我们用完 Store，可以关闭底层文件。

## 总结

### 包结构

这一章意味着我们想创建两个应用，复用我们目前为止写的领域代码。为了做到这一点，我们需要更新我们的包结构，让我们的各个 `main` 在分开的文件夹里。

通过这样做，我们因未导出的值遇到了集成问题，这进一步证明了以小"切片"工作并经常集成的价值。

我们学到了 `mypackage_test` 怎样帮我们创建一个与其他包集成你代码时相同的测试环境，帮你捕获集成问题，并看到你的代码是否（或不！）易于使用。

### 读取用户输入

我们看到从 `os.Stdin` 读取对我们来说非常容易，因为它实现了 `io.Reader`。我们用 `bufio.Scanner` 轻松地按行读取用户输入。

### 简单的抽象带来更简单的代码复用

把 `PlayerStore` 集成到我们的新应用中几乎不费力气（一旦我们做好了包的调整），随后测试也非常容易，因为我们决定也暴露我们的 stub 版本。
