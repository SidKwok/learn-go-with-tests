# WebSockets

[**本章的所有代码可以在这里找到**](https://github.com/quii/learn-go-with-tests/tree/main/websockets)

在本章中我们将学习如何使用 WebSockets 来改进我们的应用。

## 项目回顾

我们的扑克代码库里有两个应用

* _命令行应用_。提示用户输入一局游戏的玩家数量。从那时起，它会告知玩家"盲注"金额是多少，并随时间增长。任何时候用户都可以输入 `"{Playername} wins"` 来结束游戏，并把胜者记录到 store 里。
* _Web 应用_。允许用户记录游戏的胜者并展示 league 表。它和命令行应用共享同一个 store。

## 下一步

产品负责人对命令行应用很满意，但她希望我们能把这个功能搬到浏览器里。她设想一个网页，包含一个文本框让用户输入玩家数量，提交表单后页面显示盲注金额，并在适当的时候自动更新。和命令行应用一样，用户可以宣布胜者并被保存到数据库中。

表面上看，这听起来挺简单，但和往常一样，我们必须强调以 _迭代_ 的方式编写软件。

首先我们需要支持返回 HTML。到目前为止，我们所有的 HTTP 接口要么返回纯文本，要么返回 JSON。我们 _可以_ 用我们已知的技术（毕竟它们最终都是字符串），但我们也可以使用 [html/template](https://golang.org/pkg/html/template/) 包，得到一个更整洁的方案。

我们还需要能够异步地向用户发送类似 `The blind is now *y*` 的消息，而不需要刷新浏览器。我们可以使用 [WebSockets](https://en.wikipedia.org/wiki/WebSocket) 来实现这个。

> WebSocket 是一种计算机通信协议，在单个 TCP 连接上提供全双工通信通道

考虑到我们要采用一些新技术，更应该先做最少量的有用工作，再做迭代。

因此，我们要做的第一件事是创建一个网页，里面有一个表单让用户记录胜者。我们不会用普通的表单，而是用 WebSockets 把数据发送到我们的服务器去记录。

之后我们会着手处理盲注提示，那时我们已经有了一些基础设施代码。

### JavaScript 的测试怎么办？

我们会写一些 JavaScript，但我不会展开讲怎么测试它。

测试当然是可以的，但为了简洁，我不会包含相关说明。

抱歉了各位。请游说 O'Reilly 付钱让我做一本《用测试学 JavaScript》。

## 先写测试

首先我们需要在用户访问 `/game` 时返回 HTML。

提醒一下我们 web 服务器里相关的代码

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

const jsonContentType = "application/json"

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}
```

现在我们能做的 _最简单_ 的事是检查我们 `GET /game` 时拿到 `200`。

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request, _ := http.NewRequest(http.MethodGet, "/game", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

## 尝试运行测试

```
--- FAIL: TestGame (0.00s)
=== RUN   TestGame/GET_/game_returns_200
    --- FAIL: TestGame/GET_/game_returns_200 (0.00s)
    	server_test.go:109: did not get correct status, got 404, want 200
```

## 写够代码让测试通过

我们的服务器有路由器设置，所以修起来相对简单。

给我们的路由器添加

```go
router.Handle("/game", http.HandlerFunc(p.game))
```

然后写 `game` 方法

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}
```

## 重构

服务器代码已经没什么问题了，因为我们能很容易地把更多代码插进现有的良好分解的代码中。

我们可以稍微整理一下测试，添加一个辅助函数 `newGameRequest` 来构造对 `/game` 的请求。试着自己写一下。

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request := newGameRequest()
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response, http.StatusOK)
	})
}
```

你也会注意到我把 `assertStatus` 改成接受 `response` 而不是 `response.Code`，因为我觉得这样读起来更顺。

现在我们需要让接口返回一些 HTML，下面是它

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Let's play poker</title>
</head>
<body>
<section id="game">
    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>
</section>
</body>
<script type="application/javascript">

    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    if (window['WebSocket']) {
        const conn = new WebSocket('ws://' + document.location.host + '/ws')

        submitWinnerButton.onclick = event => {
            conn.send(winnerInput.value)
        }
    }
</script>
</html>
```

我们有了一个非常简单的网页

* 一个文本输入框，让用户输入胜者
* 一个按钮，他们点击它来宣布胜者。
* 一段 JavaScript，用来打开到我们服务器的 WebSocket 连接并处理提交按钮被按下的事件

`WebSocket` 在大多数现代浏览器中是内置的，所以我们不需要担心引入任何库。这个网页对老旧浏览器不工作，但在我们这个场景里这没问题。

### 我们怎么测试返回了正确的标记？

有几种方式。正如本书一直强调的，重要的是你写的测试要有足够的价值来证明它的成本。

1. 写一个基于浏览器的测试，使用 Selenium 之类的东西。这类测试是所有方法中最"真实"的，因为它们会启动某种真实的浏览器并模拟用户与之交互。这类测试能给你很高的系统能工作的信心，但比单元测试更难写、跑起来也慢得多。对我们这个产品的目的来说，这是大材小用。
2. 做精确的字符串匹配。这 _可以_ 没问题，但这类测试最终会非常脆弱。一旦有人改动了标记，你就会有一个失败的测试，但实际并 _没有真正坏掉_ 任何东西。
3. 检查我们调用了正确的模板。我们会用标准库里的模板库来生成 HTML（很快会讨论），我们可以注入用来生成 HTML 的 _东西_，并 spy 它的调用，检查我们做对了。这会对代码设计有一些影响，但其实没测出多少东西，只是确认我们用正确的模板文件调用了它而已。鉴于我们项目里只会有一个模板，这里失败的可能性看起来很低。

所以在《Learn Go with Tests》这本书里，我们第一次决定不写测试。

把标记放在一个名为 `game.html` 的文件里。

接着把我们刚写的接口改成下面这样

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	tmpl, err := template.ParseFiles("game.html")

	if err != nil {
		http.Error(w, fmt.Sprintf("problem loading template %s", err.Error()), http.StatusInternalServerError)
		return
	}

	tmpl.Execute(w, nil)
}
```

[`html/template`](https://golang.org/pkg/html/template/) 是 Go 用来创建 HTML 的包。在我们的例子里，我们调用 `template.ParseFiles` 并传入 html 文件的路径。假设没有错误，你接着可以 `Execute` 模板，它会把渲染结果写入一个 `io.Writer`。在我们的例子里我们想让它 `Write` 到互联网上，所以我们传入 `http.ResponseWriter`。

由于我们没有写测试，谨慎的做法是手动测试一下我们的 web 服务器，确保它如预期工作。进入 `cmd/webserver` 并运行 `main.go` 文件。访问 `http://localhost:5000/game`。

你 _应该_ 会得到一个错误，提示找不到模板。你可以把路径改成相对你的目录的，或者在 `cmd/webserver` 目录里放一份 `game.html` 的副本。我选择创建一个符号链接（`ln -s ../../game.html game.html`）指向项目根目录里的那个文件，这样我做改动后服务器运行时也能反映出来。

完成这个改动并再次运行后，你应该能看到我们的 UI。

现在我们需要测试：当我们通过 WebSocket 连接收到一个字符串时，我们把它当作一局游戏的胜者来记录。

## 先写测试

我们第一次要使用一个外部库，这样我们可以处理 WebSockets。

运行 `go get github.com/gorilla/websocket`

这会拉取出色的 [Gorilla WebSocket](https://github.com/gorilla/websocket) 库的代码。现在我们可以为新需求更新我们的测试。

```go
t.Run("when we get a message over a websocket it is a winner of a game", func(t *testing.T) {
	store := &StubPlayerStore{}
	winner := "Ruth"
	server := httptest.NewServer(NewPlayerServer(store))
	defer server.Close()

	wsURL := "ws" + strings.TrimPrefix(server.URL, "http") + "/ws"

	ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", wsURL, err)
	}
	defer ws.Close()

	if err := ws.WriteMessage(websocket.TextMessage, []byte(winner)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}

	AssertPlayerWin(t, store, winner)
})
```

确认你已经 import 了 `websocket` 库。我的 IDE 自动帮我做了，你的应该也会。

要测试浏览器的行为，我们必须自己打开一个 WebSocket 连接并往里写。

之前针对服务器的测试只是调用服务器上的方法，但现在我们需要和服务器有一个持久连接。为此我们使用 `httptest.NewServer`，它接受一个 `http.Handler` 并把它跑起来监听连接。

使用 `websocket.DefaultDialer.Dial` 我们尝试拨号到我们的服务器，然后尝试发送带有 `winner` 的消息。

最后，我们对 player store 做断言，检查胜者是否被记录。

## 尝试运行测试

```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:124: could not open a ws connection on ws://127.0.0.1:55838/ws websocket: bad handshake
```

我们还没有改我们的服务器接受 `/ws` 上的 WebSocket 连接，所以握手还没成功。

## 写够代码让测试通过

给我们的路由器再加一项

```go
router.Handle("/ws", http.HandlerFunc(p.webSocket))
```

然后添加我们新的 `webSocket` handler

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	upgrader.Upgrade(w, r, nil)
}
```

要接受一个 WebSocket 连接，我们需要 `Upgrade` 请求。如果你现在重新跑测试，应该会推进到下一个错误。

```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:132: got 0 calls to RecordWin want 1
```

现在我们已经打开了连接，我们要监听消息并把它记录为胜者。

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	conn, _ := upgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

（是的，我们现在忽略了很多错误！）

`conn.ReadMessage()` 阻塞等待连接上来一条消息。一旦拿到一条，我们就用它来 `RecordWin`。这之后会关闭 WebSocket 连接。

如果你试着运行测试，它仍然会失败。

问题是时序。我们的 WebSocket 连接读取消息并记录胜者之间存在延迟，而我们的测试在它发生前就结束了。你可以在最终断言前放一小段 `time.Sleep` 来验证这一点。

我们暂且这么做，但要承认在测试里放任意 sleep **是非常糟糕的实践**。

```go
time.Sleep(10 * time.Millisecond)
AssertPlayerWin(t, store, winner)
```

## 重构

我们在服务器代码和测试代码里都犯了不少"罪"才让这个测试工作起来，但记住这是我们能采用的最简单的工作方式。

我们有了肮脏、丑陋、_能工作的_ 软件，并且有测试支撑，所以我们现在可以放心地把它弄整洁，知道我们不会意外搞坏什么。

让我们从服务器代码开始。

我们可以把 `upgrader` 移到包里的私有值，因为我们不需要每次 WebSocket 连接请求都重新声明它

```go
var wsUpgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
}

func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

我们对 `template.ParseFiles("game.html")` 的调用会在每次 `GET /game` 时运行，这意味着即使我们不需要重新解析模板，每个请求都会去访问文件系统。让我们重构代码，让它在 `NewPlayerServer` 里只解析一次模板。我们必须让这个函数现在能返回一个错误，以应对从磁盘获取模板或解析它时可能出现的问题。

下面是 `PlayerServer` 相关的改动

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
}

const htmlTemplatePath = "game.html"

func NewPlayerServer(store PlayerStore) (*PlayerServer, error) {
	p := new(PlayerServer)

	tmpl, err := template.ParseFiles(htmlTemplatePath)

	if err != nil {
		return nil, fmt.Errorf("problem opening %s %v", htmlTemplatePath, err)
	}

	p.template = tmpl
	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))
	router.Handle("/game", http.HandlerFunc(p.game))
	router.Handle("/ws", http.HandlerFunc(p.webSocket))

	p.Handler = router

	return p, nil
}

func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	p.template.Execute(w, nil)
}
```

我们改了 `NewPlayerServer` 的签名，所以现在有编译问题。试着自己修复，或者参考源码。

测试代码方面，我做了一个辅助函数 `mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer`，这样我可以把错误处理的噪声隐藏在测试外。

```go
func mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer {
	server, err := NewPlayerServer(store)
	if err != nil {
		t.Fatal("problem creating player server", err)
	}
	return server
}
```

类似地，我创建了另一个辅助函数 `mustDialWS`，这样在创建 WebSocket 连接时我能隐藏讨厌的错误处理噪声。

```go
func mustDialWS(t *testing.T, url string) *websocket.Conn {
	ws, _, err := websocket.DefaultDialer.Dial(url, nil)

	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", url, err)
	}

	return ws
}
```

最后，在测试代码中我们可以创建一个辅助函数来整理发送消息的部分

```go
func writeWSMessage(t testing.TB, conn *websocket.Conn, message string) {
	t.Helper()
	if err := conn.WriteMessage(websocket.TextMessage, []byte(message)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}
}
```

现在测试通过了，试着运行服务器并在 `/game` 中宣布一些胜者。你应该能在 `/league` 看到他们被记录。记住每次我们拿到一个胜者就 _关闭连接_，所以你需要刷新页面才能再次打开连接。

我们做了一个简单的 web 表单，让用户可以记录一局游戏的胜者。让我们继续迭代，让用户可以通过提供玩家数量来开始一局游戏，服务器会随时间向客户端推送消息告知盲注金额。

首先更新 `game.html` 来更新我们客户端代码以满足新需求

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Lets play poker</title>
</head>
<body>
<section id="game">
    <div id="game-start">
        <label for="player-count">Number of players</label>
        <input type="number" id="player-count"/>
        <button id="start-game">Start</button>
    </div>

    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>

    <div id="blind-value"/>
</section>

<section id="game-end">
    <h1>Another great game of poker everyone!</h1>
    <p><a href="/league">Go check the league table</a></p>
</section>

</body>
<script type="application/javascript">
    const startGame = document.getElementById('game-start')

    const declareWinner = document.getElementById('declare-winner')
    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    const blindContainer = document.getElementById('blind-value')

    const gameContainer = document.getElementById('game')
    const gameEndContainer = document.getElementById('game-end')

    declareWinner.hidden = true
    gameEndContainer.hidden = true

    document.getElementById('start-game').addEventListener('click', event => {
        startGame.hidden = true
        declareWinner.hidden = false

        const numberOfPlayers = document.getElementById('player-count').value

        if (window['WebSocket']) {
            const conn = new WebSocket('ws://' + document.location.host + '/ws')

            submitWinnerButton.onclick = event => {
                conn.send(winnerInput.value)
                gameEndContainer.hidden = false
                gameContainer.hidden = true
            }

            conn.onclose = evt => {
                blindContainer.innerText = 'Connection closed'
            }

            conn.onmessage = evt => {
                blindContainer.innerText = evt.data
            }

            conn.onopen = function () {
                conn.send(numberOfPlayers)
            }
        }
    })
</script>
</html>
```

主要改动是引入了一段输入玩家数量的部分，以及一段显示盲注金额的部分。我们有一点逻辑根据游戏阶段显示/隐藏 UI。

任何通过 `conn.onmessage` 收到的消息我们都假定为盲注提示，因此把 `blindContainer.innerText` 相应地设置好。

我们要怎么发送盲注提示呢？在上一章我们引入了 `Game` 的概念，这样我们的 CLI 代码就可以调用 `Game`，其他的事情都会被处理好，包括调度盲注提示。这是一个不错的关注点分离。

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

当用户在 CLI 中被提示输入玩家数量时，它会 `Start` 游戏，从而启动盲注提示；当用户宣布胜者时，它会 `Finish`。这就是我们现在的需求，只是输入方式不同；所以如果可以的话我们应该尽量复用这个概念。

我们 `Game` 的"真实"实现是 `TexasHoldem`

```go
type TexasHoldem struct {
	alerter BlindAlerter
	store   PlayerStore
}
```

通过传入一个 `BlindAlerter`，`TexasHoldem` 可以把盲注提示安排发送到 _任何地方_

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

提醒一下，下面是我们在 CLI 中使用的 `BlindAlerter` 实现。

```go
func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

这在 CLI 里有效是因为 _我们总是想把提示发送到 `os.Stdout`_，但对我们的 web 服务器不行。每个请求我们都会获得一个新的 `http.ResponseWriter`，然后再把它升级到 `*websocket.Conn`。所以在构造依赖时我们没法知道提示需要去哪里。

因此我们需要修改 `BlindAlerter.ScheduleAlertAt`，让它接收一个目的地参数，这样我们就能在 web 服务器中复用它。

打开 `blind_alerter.go` 并把 `io.Writer` 参数加进去

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int, to io.Writer)
}

type BlindAlerterFunc func(duration time.Duration, amount int, to io.Writer)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int, to io.Writer) {
	a(duration, amount, to)
}
```

`StdoutAlerter` 这个名字已经不符合我们的新模型了，所以把它重命名为 `Alerter`

```go
func Alerter(duration time.Duration, amount int, to io.Writer) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(to, "Blind is now %d\n", amount)
	})
}
```

如果你试着编译，`TexasHoldem` 会编译失败，因为它调用 `ScheduleAlertAt` 没有传目的地，为了让代码 _暂时_ 能编译过，把它硬编码成 `os.Stdout`。

试着运行测试，它们会失败，因为 `SpyBlindAlerter` 不再实现 `BlindAlerter` 了，通过更新 `ScheduleAlertAt` 的签名来修复，运行测试应该又是绿色。

`TexasHoldem` 知道盲注提示要发到哪里其实没什么道理。让我们更新 `Game`，这样当你开始一局游戏时就要声明 _提示_ 应当去哪里。

```go
type Game interface {
	Start(numberOfPlayers int, alertsDestination io.Writer)
	Finish(winner string)
}
```

让编译器告诉你需要修什么。改动并不大：

* 更新 `TexasHoldem` 让它正确实现 `Game`
* 在 `CLI` 里我们开始游戏时，传入我们的 `out` 属性（`cli.game.Start(numberOfPlayers, cli.out)`）
* 在 `TexasHoldem` 的测试里，我用 `game.Start(5, io.Discard)` 来修复编译问题，并把提示输出配置为丢弃

如果你都做对了，应该一切都是绿色的！现在我们可以试着在 `Server` 里使用 `Game`。

## 先写测试

`CLI` 和 `Server` 的需求是一样的！只是交付机制不同。

让我们看看 `CLI` 的测试以获取灵感。

```go
t.Run("start game with 3 players and finish game with 'Chris' as winner", func(t *testing.T) {
	game := &GameSpy{}

	out := &bytes.Buffer{}
	in := userSends("3", "Chris wins")

	poker.NewCLI(in, out, game).PlayPoker()

	assertMessagesSentToUser(t, out, poker.PlayerPrompt)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, "Chris")
})
```

看起来我们应该可以用 `GameSpy` 测试驱动出一个类似的结果。

把旧的 websocket 测试替换成下面这样

```go
t.Run("start a game with 3 players and declare Ruth the winner", func(t *testing.T) {
	game := &poker.GameSpy{}
	winner := "Ruth"
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(10 * time.Millisecond)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)
})
```

* 如前所述，我们创建一个 spy `Game` 并把它传给 `mustMakePlayerServer`（请确保更新该辅助函数以支持这个）。
* 然后我们发送 web socket 消息来玩一局游戏。
* 最后我们断言游戏被以我们期望的方式开始和结束。

## 尝试运行测试

你会在其他测试中遇到一堆和 `mustMakePlayerServer` 有关的编译错误。引入一个非导出变量 `dummyGame`，并在所有编译失败的测试中使用它

```go
var (
	dummyGame = &GameSpy{}
)
```

最后的错误是我们在尝试把 `Game` 传给 `NewPlayerServer`，但它还不支持

```
./server_test.go:21:38: too many arguments in call to "github.com/quii/learn-go-with-tests/WebSockets/v2".NewPlayerServer
	have ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore, "github.com/quii/learn-go-with-tests/WebSockets/v2".Game)
	want ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore)
```

## 写最少量的代码让测试运行起来，并检查失败的测试输出

先把它作为参数加上去，仅仅为了让测试能跑起来

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error)
```

终于！

```
=== RUN   TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.01s)
    --- FAIL: TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner (0.01s)
    	server_test.go:146: wanted Start called with 3 but got 0
    	server_test.go:147: expected finish called with 'Ruth' but got ''
FAIL
```

## 写够代码让测试通过

我们需要把 `Game` 作为字段加到 `PlayerServer` 里，这样它在收到请求时可以使用它。

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
	game     Game
}
```

（我们已经有一个叫 `game` 的方法了，所以把它重命名为 `playGame`）

接下来在我们的构造函数里赋值

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error) {
	p := new(PlayerServer)

	tmpl, err := template.ParseFiles(htmlTemplatePath)

	if err != nil {
		return nil, fmt.Errorf("problem opening %s %v", htmlTemplatePath, err)
	}

	p.game = game

	// 等等
}
```

现在我们可以在 `webSocket` 里使用 `Game` 了。

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)

	_, numberOfPlayersMsg, _ := conn.ReadMessage()
	numberOfPlayers, _ := strconv.Atoi(string(numberOfPlayersMsg))
	p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!

	_, winner, _ := conn.ReadMessage()
	p.game.Finish(string(winner))
}
```

万岁！测试通过了。

我们 _暂时还不_ 把盲注消息发送到任何地方，因为我们需要思考一下怎么做。当我们调用 `game.Start` 时，我们传入 `io.Discard`，它会丢弃任何写入它的消息。

现在启动 web 服务器。你需要更新 `main.go` 来给 `PlayerServer` 传入一个 `Game`

```go
func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.NewTexasHoldem(poker.BlindAlerterFunc(poker.Alerter), store)

	server, err := poker.NewPlayerServer(store, game)

	if err != nil {
		log.Fatalf("problem creating player server %v", err)
	}

	log.Fatal(http.ListenAndServe(":5000", server))
}
```

抛开我们还没收到盲注提示这一事实不谈，应用是能工作的！我们已经设法把 `Game` 与 `PlayerServer` 复用起来，它处理了所有细节。一旦我们搞清楚怎么把盲注提示送到 web sockets 而不是丢弃它们，整体就 _应该_ 能工作。

不过在那之前，先整理一下代码。

## 重构

我们使用 WebSockets 的方式相当朴素，错误处理也很naive，所以我想把它封装到一个类型里，从而把这种凌乱从服务器代码中移除。我们以后可能会再回来看它，但现在这样会让事情整洁一些

```go
type playerServerWS struct {
	*websocket.Conn
}

func newPlayerServerWS(w http.ResponseWriter, r *http.Request) *playerServerWS {
	conn, err := wsUpgrader.Upgrade(w, r, nil)

	if err != nil {
		log.Printf("problem upgrading connection to WebSockets %v\n", err)
	}

	return &playerServerWS{conn}
}

func (w *playerServerWS) WaitForMsg() string {
	_, msg, err := w.ReadMessage()
	if err != nil {
		log.Printf("error reading from websocket %v\n", err)
	}
	return string(msg)
}
```

现在服务器代码简化了一些

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

一旦我们搞清楚怎么不丢弃盲注消息，我们就完工了。

### 我们 _不_ 写测试！

有时当我们不确定怎么做某件事时，最好就是玩一玩、试一试！先确保你的工作已经提交，因为一旦我们想清楚了做法，应该用测试来驱动它。

我们现在有问题的代码行是

```go
p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!
```

我们需要传入一个 `io.Writer`，让 game 把盲注提示写进去。

如果我们能传入之前的 `playerServerWS` 不就好了吗？它是我们 WebSocket 的封装，所以 _感觉_ 我们应该能把它送给 `Game` 来发送消息。

试一下：

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)
	//等等...
}
```

编译器抱怨

```
./server.go:71:14: cannot use ws (type *playerServerWS) as type io.Writer in argument to p.game.Start:
	*playerServerWS does not implement io.Writer (missing Write method)
```

显然要做的事就是让 `playerServerWS` _实现_ `io.Writer`。为此我们使用底层的 `*websocket.Conn` 调用 `WriteMessage`，把消息发送到 websocket

```go
func (w *playerServerWS) Write(p []byte) (n int, err error) {
	err = w.WriteMessage(websocket.TextMessage, p)

	if err != nil {
		return 0, err
	}

	return len(p), nil
}
```

这看起来太简单了！试着运行应用，看看能不能工作。

事先编辑 `TexasHoldem`，让盲注递增的时间间隔短一些，这样你就能看到效果

```go
blindIncrement := time.Duration(5+numberOfPlayers) * time.Second // （而不是一分钟）
```

你应该能看到它在工作！盲注金额像变魔术一样在浏览器里递增。

现在让我们把代码回滚，思考怎么测试它。为了 _实现_ 它，我们做的就是把传给 `StartGame` 的 `io.Discard` 换成了 `playerServerWS`，所以你可能会觉得我们应该 spy 这次调用来验证它工作。

Spy 很棒，能帮我们检查实现细节，但只要可以，我们应该总是倾向于测试 _真实_ 行为，因为当你决定重构时，往往是 spy 测试开始失败，因为它们通常检查的是你想改的实现细节。

我们的测试目前会打开一个到运行中服务器的 websocket 连接，并发送消息让它做事。同样地，我们也应该可以测试我们的服务器通过 websocket 连接发送回来的消息。

## 先写测试

我们会编辑现有的测试。

目前，当你调用 `Start` 时，`GameSpy` 不会向 `out` 发送任何数据。我们应该改成可以配置它发送一条预先准备好的消息，然后我们就可以检查那条消息被发送到 websocket。这应该让我们对配置正确性有信心，同时仍然在锻炼我们想要的真实行为。

```go
type GameSpy struct {
	StartCalled     bool
	StartCalledWith int
	BlindAlert      []byte

	FinishedCalled   bool
	FinishCalledWith string
}
```

添加 `BlindAlert` 字段。

更新 `GameSpy` 的 `Start`，让它把预设的消息发送到 `out`。

```go
func (g *GameSpy) Start(numberOfPlayers int, out io.Writer) {
	g.StartCalled = true
	g.StartCalledWith = numberOfPlayers
	out.Write(g.BlindAlert)
}
```

这意味着当我们驱动 `PlayerServer` 调用 `Start` 游戏时，如果一切正确，最终消息会通过 websocket 发送出去。

最后，我们可以更新测试

```go
t.Run("start a game with 3 players, send some blind alerts down WS and declare Ruth the winner", func(t *testing.T) {
	wantedBlindAlert := "Blind is 100"
	winner := "Ruth"

	game := &GameSpy{BlindAlert: []byte(wantedBlindAlert)}
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(10 * time.Millisecond)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)

	_, gotBlindAlert, _ := ws.ReadMessage()

	if string(gotBlindAlert) != wantedBlindAlert {
		t.Errorf("got blind alert %q, want %q", string(gotBlindAlert), wantedBlindAlert)
	}
})
```

* 我们加了一个 `wantedBlindAlert`，并配置我们的 `GameSpy` 在 `Start` 被调用时把它发送到 `out`。
* 我们希望它通过 websocket 连接发送出去，所以我们加了一个 `ws.ReadMessage()` 的调用，等待消息送来再检查它是否是我们期望的那一条。

## 尝试运行测试

你会发现测试永远挂着。这是因为 `ws.ReadMessage()` 会阻塞直到收到消息，但消息永远不会来。

## 写最少量的代码让测试运行起来，并检查失败的测试输出

我们绝不应该有挂住的测试，所以让我们引入一种方式，处理那些我们想要超时的代码。

```go
func within(t testing.TB, d time.Duration, assert func()) {
	t.Helper()

	done := make(chan struct{}, 1)

	go func() {
		assert()
		done <- struct{}{}
	}()

	select {
	case <-time.After(d):
		t.Error("timed out")
	case <-done:
	}
}
```

`within` 做的事是把一个函数 `assert` 作为参数接收，然后在一个 goroutine 中运行它。当函数完成时，它会通过 `done` channel 发出完成信号。

与此同时，我们使用一个 `select` 语句，让我们可以等待某个 channel 发来消息。从这里开始，这就是一个 `assert` 函数和 `time.After` 之间的赛跑，`time.After` 会在指定时长过去后发出信号。

最后，我为我们的断言做了一个辅助函数，让事情更整洁一些

```go
func assertWebsocketGotMsg(t *testing.T, ws *websocket.Conn, want string) {
	_, msg, _ := ws.ReadMessage()
	if string(msg) != want {
		t.Errorf(`got "%s", want "%s"`, string(msg), want)
	}
}
```

下面是测试现在的样子

```go
t.Run("start a game with 3 players, send some blind alerts down WS and declare Ruth the winner", func(t *testing.T) {
	wantedBlindAlert := "Blind is 100"
	winner := "Ruth"

	game := &GameSpy{BlindAlert: []byte(wantedBlindAlert)}
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(tenMS)

	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)
	within(t, tenMS, func() { assertWebsocketGotMsg(t, ws, wantedBlindAlert) })
})
```

现在如果你运行测试……

```
=== RUN   TestGame
=== RUN   TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.02s)
    --- FAIL: TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner (0.02s)
    	server_test.go:143: timed out
    	server_test.go:150: got "", want "Blind is 100"
```

## 写够代码让测试通过

最后，我们现在可以改服务器代码了，让它在游戏开始时把我们的 WebSocket 连接发送给 game

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

## 重构

服务器代码改动很小，所以这里没什么要改的，但测试代码里仍然有一处 `time.Sleep` 调用，因为我们必须等待服务器异步完成它的工作。

我们可以重构辅助函数 `assertGameStartedWith` 和 `assertFinishCalledWith`，让它们在失败前在短时间内重试断言。

下面是 `assertFinishCalledWith` 的写法，你可以对另一个辅助函数采用同样的方法。

```go
func assertFinishCalledWith(t testing.TB, game *GameSpy, winner string) {
	t.Helper()

	passed := retryUntil(500*time.Millisecond, func() bool {
		return game.FinishCalledWith == winner
	})

	if !passed {
		t.Errorf("expected finish called with %q but got %q", winner, game.FinishCalledWith)
	}
}
```

下面是 `retryUntil` 的定义

```go
func retryUntil(d time.Duration, f func() bool) bool {
	deadline := time.Now().Add(d)
	for time.Now().Before(deadline) {
		if f() {
			return true
		}
	}
	return false
}
```

## 总结

我们的应用现在完整了。一局扑克游戏可以通过 web 浏览器开始，用户会随时间通过 WebSockets 被告知盲注金额。游戏结束时他们可以记录胜者，胜者会通过我们几章前写的代码被持久化。玩家可以通过网站的 `/league` 接口找出谁是最棒的（或最幸运的）扑克玩家。

一路走来我们犯过错误，但凭借 TDD 流程，我们离能工作的软件从未很远。我们能够自由地持续迭代和实验。

最后一章会回顾我们采用的方法、最终到达的设计，并收尾一些遗留的问题。

我们在本章中讲了几件事

### WebSockets

* 一种方便的、在客户端和服务器之间发送消息的方式，且不要求客户端不停地轮询服务器。我们的客户端代码和服务器代码都很简单。
* 测试很容易，但你必须留意测试的异步特性

### 处理可能延迟或永远不会完成的代码

* 创建辅助函数来重试断言并加上超时。
* 我们可以用 goroutine 让断言不阻塞任何东西，再用 channel 让它们发出"我已经完成"或者"未完成"的信号。
* `time` 包有一些有用的函数，它们也通过 channel 发出关于时间的事件信号，所以我们可以设置超时
