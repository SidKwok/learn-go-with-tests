# JSON、路由与嵌入

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/json)**

[在上一章](http-server.md) 我们创建了一个 web 服务器，用来存储玩家赢了多少场游戏。

我们的产品负责人有了新需求：增加一个名为 `/league` 的新接口，返回所有已存储玩家的列表。她希望以 JSON 形式返回。

## 这是我们目前的代码

```go
// server.go
package main

import (
	"fmt"
	"net/http"
	"strings"
)

type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}

func (p *PlayerServer) showScore(w http.ResponseWriter, player string) {
	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter, player string) {
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

```go
// in_memory_player_store.go
package main

func NewInMemoryPlayerStore() *InMemoryPlayerStore {
	return &InMemoryPlayerStore{map[string]int{}}
}

type InMemoryPlayerStore struct {
	store map[string]int
}

func (i *InMemoryPlayerStore) RecordWin(name string) {
	i.store[name]++
}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return i.store[name]
}

```

```go
// main.go
package main

import (
	"log"
	"net/http"
)

func main() {
	server := &PlayerServer{NewInMemoryPlayerStore()}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

对应的测试可以在本章顶部的链接里找到。

我们先从制作 league 表接口开始。

## 先写测试

我们会扩展现有的测试套件，因为我们已经有一些有用的测试函数和一个假的 `PlayerStore` 可以用。

```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := &PlayerServer{&store}

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

在操心实际分数和 JSON 之前，我们会让改动尽量小，按计划朝目标迭代。最简单的开始就是检查我们能命中 `/league` 并拿回 `OK`。

## 尝试运行测试

```
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:101: status code is wrong: got 404, want 200
FAIL
FAIL	playerstore	0.221s
FAIL
```

我们的 `PlayerServer` 返回了 `404 Not Found`，就好像我们在尝试获取一个未知玩家的胜场数。看一下 `server.go` 里 `ServeHTTP` 的实现，我们意识到它总是假设被调用时 URL 指向一个特定玩家：

```go
player := strings.TrimPrefix(r.URL.Path, "/players/")
```

在上一章我们提到过这是一种相当朴素的路由方式。我们的测试正确地告诉我们：需要一种概念来处理不同的请求路径。

## 写够代码让测试通过

Go 内置了一个路由机制叫 [`ServeMux`](https://golang.org/pkg/net/http/#ServeMux)（请求多路复用器），它让你可以把 `http.Handler` 挂到特定的请求路径上。

让我们先犯点"小错"，用最快的方式让测试通过，知道一旦测试通过我们就可以安全地重构。

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()

	router.Handle("/league", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	router.Handle("/players/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		player := strings.TrimPrefix(r.URL.Path, "/players/")

		switch r.Method {
		case http.MethodPost:
			p.processWin(w, player)
		case http.MethodGet:
			p.showScore(w, player)
		}
	}))

	router.ServeHTTP(w, r)
}
```

- 当请求开始时，我们创建一个路由器，然后告诉它对路径 `x` 使用处理器 `y`。
- 所以对于我们的新接口，我们用 `http.HandlerFunc` 加一个 _匿名函数_，在 `/league` 被请求时调用 `w.WriteHeader(http.StatusOK)` 来让我们的新测试通过。
- 对于 `/players/` 路由，我们直接把代码剪切粘贴到另一个 `http.HandlerFunc` 里。
- 最后，我们通过调用新路由器的 `ServeHTTP` 来处理进入的请求（注意 `ServeMux` _也是_ 一个 `http.Handler`？）

测试现在应该能通过。

## 重构

`ServeHTTP` 看起来挺大的，我们可以把处理器重构成独立的方法，让事情分离一些。

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	router.ServeHTTP(w, r)
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) playersHandler(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}
```

每次请求来的时候才设置路由器再调用它，挺奇怪的（也低效）。理想情况下我们想要一个 `NewPlayerServer` 函数，它接受我们的依赖并一次性完成创建路由器的工作。之后每个请求只需要使用这一份路由器实例。

```go
//server.go
type PlayerServer struct {
	store  PlayerStore
	router *http.ServeMux
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := &PlayerServer{
		store,
		http.NewServeMux(),
	}

	p.router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	p.router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	return p
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	p.router.ServeHTTP(w, r)
}
```

- `PlayerServer` 现在需要存储一个路由器。
- 我们把路由的创建从 `ServeHTTP` 移到了 `NewPlayerServer` 里，这样它只需要做一次，而不是每个请求都做一次。
- 你需要把所有原来用 `PlayerServer{&store}` 的测试代码和生产代码都改成 `NewPlayerServer(&store)`。

### 最后一次重构

试着把代码改成下面这样。

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

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

然后在 `server_test.go`、`server_integration_test.go` 和 `main.go` 中把 `server := &PlayerServer{&store}` 替换成 `server := NewPlayerServer(&store)`。

最后确认 **删除** `func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request)`，因为它已经不需要了！

## 嵌入（Embedding）

我们改了 `PlayerServer` 的第二个属性，把命名属性 `router http.ServeMux` 移除，换成了 `http.Handler`；这就叫 _嵌入_。

> Go 不提供典型的、由类型驱动的子类化概念，但它确实有"借用"实现的能力，即把类型嵌入到结构体或接口中。

[Effective Go - Embedding](https://golang.org/doc/effective_go.html#embedding)

这意味着我们的 `PlayerServer` 现在拥有了 `http.Handler` 拥有的所有方法，也就是只有 `ServeHTTP`。

为了"填上" `http.Handler`，我们把它赋值为我们在 `NewPlayerServer` 里创建的 `router`。我们之所以可以这么做，是因为 `http.ServeMux` 有 `ServeHTTP` 方法。

这让我们可以删掉自己的 `ServeHTTP` 方法，因为我们已经通过嵌入的类型暴露了一个。

嵌入是一个非常有意思的语言特性。你可以把它和接口一起用，组合出新的接口。

```go
type Animal interface {
	Eater
	Sleeper
}
```

你也可以把它和具体类型一起用，不只是接口。如你所料，如果你嵌入一个具体类型，你将能访问它所有的公开方法和字段。

### 有什么坏处吗？

嵌入类型时必须小心，因为你会暴露被嵌入类型所有的公开方法和字段。在我们的案例里这没问题，因为我们嵌入的只是我们想要暴露的那个 _接口_（`http.Handler`）。

如果我们偷懒，嵌入了 `http.ServeMux`（具体类型），它仍然能工作 _但是_ `PlayerServer` 的使用者就能给我们的服务器添加新路由了，因为 `Handle(path, handler)` 会变成公开的。

**当你嵌入类型时，请认真思考它对你公共 API 的影响。**

误用嵌入并最终污染 API、暴露类型内部细节，是 _非常_ 常见的错误。

现在我们重构了应用结构，可以轻松地添加新路由，也开始有了 `/league` 接口。我们现在需要让它返回一些有用的信息。

我们应该返回类似这样的 JSON。

```json
[
   {
      "Name":"Bill",
      "Wins":10
   },
   {
      "Name":"Alice",
      "Wins":15
   }
]
```

## 先写测试

我们先尝试把响应解析成有意义的内容。

```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := NewPlayerServer(&store)

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

### 为什么不直接测试 JSON 字符串？

你可能会说更简单的初始步骤就是断言响应体是某个特定的 JSON 字符串。

根据我的经验，针对 JSON 字符串的断言测试有以下问题。

- *脆弱*。如果你改变了数据模型，你的测试就会失败。
- *难以调试*。当比较两个 JSON 字符串时，理解实际问题在哪可能很棘手。
- *意图不明*。虽然输出应该是 JSON，但真正重要的是数据具体是什么，而不是它如何被编码。
- *重复测试标准库*。没必要去测试标准库怎么输出 JSON 的，它已经被测过了。不要测试别人的代码。

我们应该把 JSON 解析成与我们要测试相关的数据结构。

### 数据建模

根据这个 JSON 数据模型，我们似乎需要一个带几个字段的 `Player` 数组，所以我们创建了一个新类型来承载这个。

```go
//server.go
type Player struct {
	Name string
	Wins int
}
```

### JSON 解码

```go
//server_test.go
var got []Player
err := json.NewDecoder(response.Body).Decode(&got)
```

要把 JSON 解析到我们的数据模型里，我们从 `encoding/json` 包创建一个 `Decoder`，然后调用它的 `Decode` 方法。要创建一个 `Decoder`，它需要一个 `io.Reader` 来读取，本例里就是我们响应 spy 的 `Body`。

`Decode` 接收我们要解码进去的目标的地址，所以前一行我们声明了一个空的 `Player` 切片。

解析 JSON 可能会失败，所以 `Decode` 可能返回一个 `error`。如果失败了就没必要继续测试，所以我们检查错误，并在出错时用 `t.Fatalf` 停止测试。注意我们把响应体也和错误一起打印了出来，因为对运行测试的人来说，看到无法解析的字符串是什么很重要。

## 尝试运行测试

```
=== RUN   TestLeague/it_returns_200_on_/league
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:107: Unable to parse response from server '' into slice of Player, 'unexpected end of JSON input'
```

我们的接口目前没有返回响应体，所以无法被解析为 JSON。

## 写够代码让测试通过

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	leagueTable := []Player{
		{"Chris", 20},
	}

	json.NewEncoder(w).Encode(leagueTable)

	w.WriteHeader(http.StatusOK)
}
```

测试现在通过了。

### 编码与解码

注意标准库里漂亮的对称性。

- 要创建一个 `Encoder`，你需要一个 `io.Writer`，而 `http.ResponseWriter` 实现了它。
- 要创建一个 `Decoder`，你需要一个 `io.Reader`，而我们响应 spy 的 `Body` 字段实现了它。

贯穿全书我们用过 `io.Writer`，这又一次展示了它在标准库里的普遍性，以及很多库是如何方便地与它协作的。

## 重构

把"获取 `leagueTable`"的关注点和处理器分开会更好，因为我们知道很快就不会硬编码它了。

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.getLeagueTable())
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) getLeagueTable() []Player {
	return []Player{
		{"Chris", 20},
	}
}
```

接下来，我们想扩展测试，让我们能精确控制要返回什么数据。

## 先写测试

我们可以更新测试，断言 league 表里包含我们将要在 store 里 stub 的一些玩家。

更新 `StubPlayerStore`，让它能存一个 league，也就是一个 `Player` 切片。我们会把期望的数据放在那儿。

```go
//server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}
```

接下来，更新当前的测试，把一些玩家放进 stub 的 league 属性里，并断言它们能从我们的服务器返回。

```go
//server_test.go
func TestLeague(t *testing.T) {

	t.Run("it returns the league table as JSON", func(t *testing.T) {
		wantedLeague := []Player{
			{"Cleo", 32},
			{"Chris", 20},
			{"Tiest", 14},
		}

		store := StubPlayerStore{nil, nil, wantedLeague}
		server := NewPlayerServer(&store)

		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)

		if !reflect.DeepEqual(got, wantedLeague) {
			t.Errorf("got %v want %v", got, wantedLeague)
		}
	})
}
```

## 尝试运行测试

```
./server_test.go:33:3: too few values in struct initializer
./server_test.go:70:3: too few values in struct initializer
```

## 写最少量的代码让测试运行起来，并检查失败的测试输出

你需要更新其他测试，因为 `StubPlayerStore` 多了一个新字段；在其他测试里把它设成 nil。

再次运行测试，你应该会看到

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: got [{Chris 20}] want [{Cleo 32} {Chris 20} {Tiest 14}]
```

## 写够代码让测试通过

我们知道数据在 `StubPlayerStore` 里，并且我们已经把它抽象成了接口 `PlayerStore`。我们需要更新这个接口，让任何传给我们 `PlayerStore` 的人都能为 league 提供数据。

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}
```

现在我们可以更新 handler 代码去调用它，而不是返回硬编码的列表。删除我们的 `getLeagueTable()` 方法，然后更新 `leagueHandler` 调用 `GetLeague()`。

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.store.GetLeague())
	w.WriteHeader(http.StatusOK)
}
```

试着运行测试。

```
# github.com/quii/learn-go-with-tests/json-and-io/v4
./main.go:9:50: cannot use NewInMemoryPlayerStore() (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_integration_test.go:11:27: cannot use store (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:36:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:74:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:106:29: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
```

编译器在抱怨，因为 `InMemoryPlayerStore` 和 `StubPlayerStore` 没有我们加到接口上的新方法。

对 `StubPlayerStore` 来说很简单，只需返回我们之前加的 `league` 字段。

```go
//server_test.go
func (s *StubPlayerStore) GetLeague() []Player {
	return s.league
}
```

提醒一下 `InMemoryStore` 是怎么实现的。

```go
//in_memory_player_store.go
type InMemoryPlayerStore struct {
	store map[string]int
}
```

虽然遍历 map "正确地"实现 `GetLeague` 也很简单，但记住我们现在只是想 _写最少量的代码让测试通过_。

所以我们暂且让编译器开心，先忍受 `InMemoryStore` 实现不完整的不适感。

```go
//in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	return nil
}
```

这其实在告诉我们 _稍后_ 我们会想测试这个，但现在先放一放。

试着运行测试，编译应该能通过，测试也应该能通过！

## 重构

测试代码没有把意图表达得很清楚，并且有很多样板代码可以重构掉。

```go
//server_test.go
t.Run("it returns the league table as JSON", func(t *testing.T) {
	wantedLeague := []Player{
		{"Cleo", 32},
		{"Chris", 20},
		{"Tiest", 14},
	}

	store := StubPlayerStore{nil, nil, wantedLeague}
	server := NewPlayerServer(&store)

	request := newLeagueRequest()
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := getLeagueFromResponse(t, response.Body)
	assertStatus(t, response.Code, http.StatusOK)
	assertLeague(t, got, wantedLeague)
})
```

下面是新的辅助函数

```go
//server_test.go
func getLeagueFromResponse(t testing.TB, body io.Reader) (league []Player) {
	t.Helper()
	err := json.NewDecoder(body).Decode(&league)

	if err != nil {
		t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", body, err)
	}

	return
}

func assertLeague(t testing.TB, got, want []Player) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}

func newLeagueRequest() *http.Request {
	req, _ := http.NewRequest(http.MethodGet, "/league", nil)
	return req
}
```

为了让我们的服务器工作，最后还有一件事要做：确保我们在响应里返回 `content-type` 响应头，这样机器才能识别我们返回的是 `JSON`。

## 先写测试

把这个断言加到现有测试里

```go
//server_test.go
if response.Result().Header.Get("content-type") != "application/json" {
	t.Errorf("response did not have content-type of application/json, got %v", response.Result().Header)
}
```

## 尝试运行测试

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: response did not have content-type of application/json, got map[Content-Type:[text/plain; charset=utf-8]]
```

## 写够代码让测试通过

更新 `leagueHandler`

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", "application/json")
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

测试应该通过。

## 重构

为 "application/json" 创建一个常量并在 `leagueHandler` 中使用它

```go
//server.go
const jsonContentType = "application/json"

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", jsonContentType)
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

再为 `assertContentType` 加一个辅助函数。

```go
//server_test.go
func assertContentType(t testing.TB, response *httptest.ResponseRecorder, want string) {
	t.Helper()
	if response.Result().Header.Get("content-type") != want {
		t.Errorf("response did not have content-type of %s, got %v", want, response.Result().Header)
	}
}
```

在测试中使用它。

```go
//server_test.go
assertContentType(t, response, jsonContentType)
```

`PlayerServer` 暂时搞定后，我们可以把注意力转向 `InMemoryPlayerStore`，因为现在如果我们尝试给产品负责人演示，`/league` 是不会工作的。

最快建立信心的方式是补充集成测试，我们可以命中新接口并检查从 `/league` 拿回的响应是否正确。

## 先写测试

我们可以用 `t.Run` 把这个测试拆开，并且可以复用我们服务器测试中的辅助函数——再次说明重构测试的重要性。

```go
//server_integration_test.go
func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := NewInMemoryPlayerStore()
	server := NewPlayerServer(store)
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	t.Run("get score", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newGetScoreRequest(player))
		assertStatus(t, response.Code, http.StatusOK)

		assertResponseBody(t, response.Body.String(), "3")
	})

	t.Run("get league", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newLeagueRequest())
		assertStatus(t, response.Code, http.StatusOK)

		got := getLeagueFromResponse(t, response.Body)
		want := []Player{
			{"Pepper", 3},
		}
		assertLeague(t, got, want)
	})
}
```

## 尝试运行测试

```
=== RUN   TestRecordingWinsAndRetrievingThem/get_league
    --- FAIL: TestRecordingWinsAndRetrievingThem/get_league (0.00s)
        server_integration_test.go:35: got [] want [{Pepper 3}]
```

## 写够代码让测试通过

`InMemoryPlayerStore` 在你调用 `GetLeague()` 时返回 `nil`，所以我们要修这个。

```go
//in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	var league []Player
	for name, wins := range i.store {
		league = append(league, Player{name, wins})
	}
	return league
}
```

我们要做的就是遍历 map，并把每一对 key/value 转成 `Player`。

测试现在应该通过。

## 总结

我们继续在 TDD 下安全地迭代我们的程序，让它通过路由可维护地支持新接口，并且现在可以为我们的消费者返回 JSON。下一章我们将讲数据持久化和 league 的排序。

我们覆盖了：

- **路由**。标准库为你提供了一个易用的类型来做路由。它完全拥抱了 `http.Handler` 接口：你给 `Handler` 分配路由，而路由器本身也是一个 `Handler`。但它没有一些你可能期望的特性，比如路径变量（例如 `/users/{id}`）。你可以自己轻松解析这些信息，但如果它成了负担，你也许会想看看其他路由库。大多数流行的路由库也遵循标准库的哲学，同样实现了 `http.Handler`。
- **类型嵌入**。我们简单触及了这项技术，你可以从 [Effective Go](https://golang.org/doc/effective_go.html#embedding) 学到更多。如果只能让你记住一件事，那就是它可以非常有用，但 _始终要思考你的公共 API，只暴露合适的部分_。
- **JSON 序列化与反序列化**。标准库让序列化和反序列化数据非常简单。它也开放配置，必要时你可以自定义这些数据转换的工作方式。
