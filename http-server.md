# HTTP Server

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/http-server)**

你被要求创建一个 web 服务器，让用户可以追踪每位玩家赢了多少场比赛。

-   `GET /players/{name}` 应返回一个数字，表示该玩家的总胜场数
-   `POST /players/{name}` 应记录该玩家的一场胜利，每次后续的 `POST` 都让计数加一

我们会遵循 TDD 的方式，尽可能快地拿到能工作的软件，然后通过小步迭代不断改进，直到拿到最终方案。采取这种方式，我们能够：

-   在任意时刻把问题域保持得足够小
-   不掉进无底洞
-   万一卡住或迷失方向，回滚也不会丢失大量工作。

## 红、绿、重构

贯穿本书，我们一直强调 TDD 的流程：写一个测试并看着它失败（红），写 _最少量_ 的代码让它通过（绿），然后重构。

写最少量代码这种纪律性很重要，它关乎 TDD 给你的安全感。你应该努力尽快从"红"的状态里走出来。

Kent Beck 是这样描述的：

> 让测试快速跑通，过程中犯什么"罪"都行。

你之所以能犯这些"罪"，是因为之后你会在测试的安全保障下进行重构。

### 如果你不这样做会怎样？

你处于红色状态的时候改动越多，就越可能引入更多没被测试覆盖的问题。

我们的目标是用小步骤、由测试驱动地、迭代式地写有用的代码，这样你就不会一连几个小时陷在某个无底洞里。

### 鸡和蛋

我们怎么逐步把这个东西搭起来？没存进什么东西就没法 `GET` 一个玩家，并且如果还没有 `GET` 接口存在，似乎也很难判断 `POST` 是否生效了。

这正是 _mock_ 大显身手的地方。

-   `GET` 需要一个 `PlayerStore` _之类_ 的东西来获取玩家的分数。它应该是一个接口，这样在测试时我们就能创建一个简单的 stub 来测试我们的代码，而不需要真的实现任何存储代码。
-   对于 `POST`，我们可以 _spy_ 它对 `PlayerStore` 的调用，确保它正确地存储了玩家。我们的保存实现不会与读取耦合在一起。
-   为了快速拿到能工作的软件，我们可以做一个非常简单的内存实现，之后再换成由我们偏好的任何存储机制支撑的实现。

## 先写测试

我们可以写一个测试，先返回一个写死的值让它通过。Kent Beck 把这叫"假装一下"（Faking it）。等我们有了一个能跑的测试，再写更多测试帮我们去掉这个常量。

通过这一步非常小的改动，我们就能在不太担心应用逻辑的前提下，把整个项目结构正确跑起来这件重要的事先搞定。

在 Go 中创建一个 web 服务器，通常会调用 [ListenAndServe](https://golang.org/pkg/net/http/#ListenAndServe)。

```go
func ListenAndServe(addr string, handler Handler) error
```

它会启动一个 web 服务器监听指定端口，对每个请求创建一个 goroutine 并把它交给 [`Handler`](https://golang.org/pkg/net/http/#Handler) 处理。

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

一个类型通过实现 `ServeHTTP` 方法来实现 Handler 接口。该方法接收两个参数：第一个是我们 _写入响应_ 的地方，第二个是发送到服务器的 HTTP 请求。

我们来创建一个名为 `server_test.go` 的文件，并为函数 `PlayerServer` 写一个测试，它接收上面这两个参数。我们传入的请求是获取一位玩家的得分，期望得到 `"20"`。
```go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		got := response.Body.String()
		want := "20"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

为了测试我们的服务器，我们需要一个 `Request` 来发起请求，并希望 _spy_ handler 写入 `ResponseWriter` 的内容。

-   我们用 `http.NewRequest` 来创建一个请求。第一个参数是请求方法，第二个是请求路径。`nil` 参数指的是请求体，本例中我们不需要设置。
-   `net/http/httptest` 已经为我们提供了一个 spy，叫做 `ResponseRecorder`，我们可以直接用它。它有许多便利的方法，可以检查写入的响应内容。

## 尝试运行测试

`./server_test.go:13:2: undefined: PlayerServer`

## 写最少量的代码让测试能跑起来，并查看失败的测试输出

编译器会帮你的，听它的话就好。

创建一个名为 `server.go` 的文件并定义 `PlayerServer`

```go
func PlayerServer() {}
```

再试一次

```
./server_test.go:13:14: too many arguments in call to PlayerServer
    have (*httptest.ResponseRecorder, *http.Request)
    want ()
```

给我们的函数加上参数

```go
import "net/http"

func PlayerServer(w http.ResponseWriter, r *http.Request) {

}
```

代码现在能编译了，测试失败

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- FAIL: TestGETPlayers/returns_Pepper's_score (0.00s)
        server_test.go:20: got '', want '20'
```

## 写足够的代码让测试通过

在 DI 那一章，我们用 `Greet` 函数浅浅接触过 HTTP 服务器。我们了解到 net/http 的 `ResponseWriter` 同时也实现了 io 的 `Writer`，所以我们可以用 `fmt.Fprint` 把字符串作为 HTTP 响应发送出去。

```go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "20")
}
```

测试现在应该通过了。

## 完成脚手架

我们想把它接入到一个应用中。这一步很重要，因为：

-   我们会有 _真实可用的软件_，我们不希望写测试只是为了写测试，看到代码真的跑起来是好事。
-   随着重构，程序结构很可能会发生变化。我们希望这些变化也能反映在我们的应用中，作为渐进式开发的一部分。

为我们的应用创建一个新的 `main.go` 文件，并放入下面的代码

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(PlayerServer)
	log.Fatal(http.ListenAndServe(":5000", handler))
}
```

到目前为止，我们的应用代码都在一个文件里，但对于较大的项目这并不是最佳实践，你会希望把不同的内容拆到不同的文件里。

要运行这个程序，执行 `go build`，它会把目录下所有 `.go` 文件构建成一个程序。然后你就可以用 `./myprogram` 执行它。

### `http.HandlerFunc`

我们之前探讨过，要做出一个服务器，需要实现的是 `Handler` 接口。_通常_ 我们的做法是创建一个 `struct`，让它实现 ServeHTTP 方法，从而实现该接口。但是 struct 的用途是承载数据，而 _目前_ 我们没有任何状态，所以创建 struct 感觉不太对劲。

[HandlerFunc](https://golang.org/pkg/net/http/#HandlerFunc) 让我们可以避开这个问题。

> HandlerFunc 类型是一个适配器，允许把普通函数当作 HTTP handler 使用。如果 f 是一个签名合适的函数，HandlerFunc(f) 就是一个调用 f 的 Handler。

```go
type HandlerFunc func(ResponseWriter, *Request)
```

从文档可以看到，类型 `HandlerFunc` 已经实现了 `ServeHTTP` 方法。
通过用它对我们的 `PlayerServer` 函数做类型转换，我们就实现了所需的 `Handler`。

### `http.ListenAndServe(":5000"...)`

`ListenAndServe` 接收一个端口和一个 `Handler`。如果出现问题，web 服务器会返回一个错误，比如端口已被监听。出于这个原因，我们用 `log.Fatal` 包住它，把错误打给用户看。

接下来我们要做的是再写 _一个_ 测试，逼我们做出一个有意义的改动，来摆脱写死的值。

## 先写测试

我们会在测试套件中再加一个子测试，尝试获取另一位玩家的分数，这会打破我们写死的方案。

```go
t.Run("returns Floyd's score", func(t *testing.T) {
	request, _ := http.NewRequest(http.MethodGet, "/players/Floyd", nil)
	response := httptest.NewRecorder()

	PlayerServer(response, request)

	got := response.Body.String()
	want := "10"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

你可能会想

> 我们肯定需要某种存储的概念，来控制哪个玩家有多少分。我们测试里的值看起来这么随意有点怪。

记住我们只是尽量以合理的范围内最小的步骤前进，所以现在我们只是先打破这个常量。

## 尝试运行测试

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- PASS: TestGETPlayers/returns_Pepper's_score (0.00s)
=== RUN   TestGETPlayers/returns_Floyd's_score
    --- FAIL: TestGETPlayers/returns_Floyd's_score (0.00s)
        server_test.go:34: got '20', want '10'
```

## 写足够的代码让测试通过

```go
//server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	if player == "Pepper" {
		fmt.Fprint(w, "20")
		return
	}

	if player == "Floyd" {
		fmt.Fprint(w, "10")
		return
	}
}
```

这个测试逼我们去真正看请求的 URL 并做出决定。所以虽然我们脑子里可能在担心 player store 和接口的事，下一步合乎逻辑的事其实是 _路由_。

如果我们一开始就动手写 store 代码，相比之下要做的改动会非常多。**这是迈向最终目标的更小一步，并且是由测试驱动的**。

我们现在抗住了使用任何路由库的诱惑，只迈出最小的一步让测试通过。

`r.URL.Path` 返回请求的路径，我们可以用 [`strings.TrimPrefix`](https://golang.org/pkg/strings/#TrimPrefix) 把 `/players/` 截掉，得到请求的玩家名。它不算很健壮，但目前够用。

## 重构

我们可以把分数获取逻辑抽出成一个函数来简化 `PlayerServer`

```go
//server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	fmt.Fprint(w, GetPlayerScore(player))
}

func GetPlayerScore(name string) string {
	if name == "Pepper" {
		return "20"
	}

	if name == "Floyd" {
		return "10"
	}

	return ""
}
```

我们也可以通过引入一些辅助函数把测试代码 DRY 化

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

不过，我们仍然不应该感到满意。让我们的服务器知道分数是不对劲的。

我们的重构已经清晰地指出了下一步该做什么。

我们把分数计算从 handler 主体里搬到了一个函数 `GetPlayerScore`。这看起来是一个合适的位置，可以用接口来分离关注点。

让我们把刚才重构出来的函数改成一个接口

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
}
```

要让我们的 `PlayerServer` 能用 `PlayerStore`，它需要持有一个对它的引用。现在似乎是个合适的时机来调整架构，把 `PlayerServer` 改成一个 `struct`。

```go
type PlayerServer struct {
	store PlayerStore
}
```

最后，我们通过给这个新的 struct 加一个方法、把现有的 handler 代码放进去，来实现 `Handler` 接口。

```go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

唯一另外要改的是，我们现在调用 `store.GetPlayerScore` 来获取分数，而不是我们之前定义的本地函数（现在可以删了）。

下面是我们 server 的完整代码

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

### 解决问题

刚才做了不少改动，我们知道测试和应用都不再能编译了，但放轻松，让编译器帮我们一步步搞定。

`./main.go:9:58: type PlayerServer is not an expression`

我们需要修改测试，改成创建一个 `PlayerServer` 的新实例，然后调用它的 `ServeHTTP` 方法。

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	server := &PlayerServer{}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

注意我们 _目前还没在_ 担心怎么做 store，我们只想尽快让编译器通过。

你应该养成一种习惯：先优先让代码能编译，然后再让测试通过。

如果在代码还没编译通过时就加更多功能（比如 stub store），我们就给自己埋下了 _更多_ 编译问题的隐患。

现在 `main.go` 也因为同样的原因不能编译。

```go
func main() {
	server := &PlayerServer{}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

终于全部能编译了，但测试失败了

```
=== RUN   TestGETPlayers/returns_the_Pepper's_score
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
    panic: runtime error: invalid memory address or nil pointer dereference
```

这是因为我们没有在测试中传入一个 `PlayerStore`。我们需要做一个 stub。

```go
//server_test.go
type StubPlayerStore struct {
	scores map[string]int
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}
```

`map` 是为我们的测试快速、轻松地做一个 stub 键值存储的好方式。现在让我们为测试创建这样一个 store，并把它传给 `PlayerServer`。

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

我们的测试现在通过了，看起来也更好了。引入 store 之后，代码 _意图_ 更清晰了。我们在告诉读者：因为 _`PlayerStore` 里有这些数据_，当你把它和 `PlayerServer` 一起使用时，应得到下面这些响应。

### 运行应用

测试通过后，要完成这次重构最后还需要做的事，是检查我们的应用是否能工作。程序应该能启动，但如果你试着访问 `http://localhost:5000/players/Pepper`，会得到一个糟糕的响应。

原因是我们没有传入 `PlayerStore`。

我们需要做一个实现，但目前比较困难，因为我们还没有任何有意义的数据要存，所以暂时只能写死。

```go
//main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return 123
}

func main() {
	server := &PlayerServer{&InMemoryPlayerStore{}}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

如果你再次运行 `go build` 并访问相同的 URL，应该会得到 `"123"`。这并不出色，但在我们能存数据之前这是最好的状态了。
我们的主程序启动了却不真正能用，这种感觉也不好。我们不得不通过手动测试才发现问题。

接下来该做什么我们有几个选择

-   处理玩家不存在的场景
-   处理 `POST /players/{name}` 的场景

虽然 `POST` 场景让我们更接近"主路径"，但我觉得先处理玩家不存在的场景更容易，因为我们已经在那个上下文里了。剩下的等会儿再处理。

## 先写测试

为已有的测试套件加一个玩家不存在的场景

```go
//server_test.go
t.Run("returns 404 on missing players", func(t *testing.T) {
	request := newGetScoreRequest("Apollo")
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := response.Code
	want := http.StatusNotFound

	if got != want {
		t.Errorf("got status %d want %d", got, want)
	}
})
```

## 尝试运行测试

```
=== RUN   TestGETPlayers/returns_404_on_missing_players
    --- FAIL: TestGETPlayers/returns_404_on_missing_players (0.00s)
        server_test.go:56: got status 200 want 404
```

## 写足够的代码让测试通过

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	w.WriteHeader(http.StatusNotFound)

	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

有时候听到 TDD 拥护者说"确保只写最少量的代码让测试通过"我会很翻白眼，因为这听起来很迂腐。

但这个场景把这一点说明得很清楚。我做了最少量的事（明知它不正确），就是对 **所有响应** 都写 `StatusNotFound`，可所有测试居然都通过了！

**通过做最少量的事让测试通过，可以暴露你测试中的盲点**。在我们这个例子里，我们没有断言：当玩家 _确实_ 存在 store 中时，应该收到 `StatusOK`。

把另外两个测试更新成对状态码也做断言，并修复代码。

下面是新的测试

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "10")
	})

	t.Run("returns 404 on missing players", func(t *testing.T) {
		request := newGetScoreRequest("Apollo")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusNotFound)
	})
}

func assertStatus(t testing.TB, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("did not get correct status, got %d, want %d", got, want)
	}
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

我们现在所有测试都在检查状态码，所以我做了一个辅助函数 `assertStatus` 来配合这件事。

现在前两个测试因为 404 而不是 200 失败了，所以我们可以修改 `PlayerServer`，只在分数为 0 时返回 not found。

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

### 存储分数

既然能从 store 中读取分数，那么能存储新分数也合情合理了。

## 先写测试

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it returns accepted on POST", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodPost, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)
	})
}
```

先简单地检查一下：当我们用 POST 命中那条特定路由时，能拿到正确的状态码。这能驱使我们把"接受不同种请求方式"的功能做出来，并和 `GET /players/{name}` 区别处理。这一步通过后，我们再开始断言 handler 与 store 之间的交互。

## 尝试运行测试

```
=== RUN   TestStoreWins/it_returns_accepted_on_POST
    --- FAIL: TestStoreWins/it_returns_accepted_on_POST (0.00s)
        server_test.go:70: did not get correct status, got 404, want 202
```

## 写足够的代码让测试通过

记住我们是在故意"犯罪"，所以一个根据请求方法的 `if` 语句就够了。

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	if r.Method == http.MethodPost {
		w.WriteHeader(http.StatusAccepted)
		return
	}

	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

## 重构

handler 现在看起来有点乱了。我们把代码拆开，让它更易读，把不同的功能抽到不同的新函数里。

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	switch r.Method {
	case http.MethodPost:
		p.processWin(w)
	case http.MethodGet:
		p.showScore(w, r)
	}

}

func (p *PlayerServer) showScore(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter) {
	w.WriteHeader(http.StatusAccepted)
}
```

这让 `ServeHTTP` 的路由部分更清晰了一些，也意味着我们之后关于存储的迭代都可以放在 `processWin` 里。

接下来，我们想检查的是当我们做 `POST /players/{name}` 时，`PlayerStore` 被告知去记录这次胜利。

## 先写测试

我们可以通过给 `StubPlayerStore` 加一个新的 `RecordWin` 方法来做到这一点，然后 spy 它的调用。

```go
//server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}
```

现在先扩展测试，检查调用次数

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it records wins when POST", func(t *testing.T) {
		request := newPostWinRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)

		if len(store.winCalls) != 1 {
			t.Errorf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
		}
	})
}

func newPostWinRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodPost, fmt.Sprintf("/players/%s", name), nil)
	return req
}
```

## 尝试运行测试

```
./server_test.go:26:20: too few values in struct initializer
./server_test.go:65:20: too few values in struct initializer
```

## 写最少量的代码让测试能跑起来，并查看失败的测试输出

我们需要更新创建 `StubPlayerStore` 的代码，因为我们加了新字段

```go
//server_test.go
store := StubPlayerStore{
	map[string]int{},
	nil,
}
```

```
--- FAIL: TestStoreWins (0.00s)
    --- FAIL: TestStoreWins/it_records_wins_when_POST (0.00s)
        server_test.go:80: got 0 calls to RecordWin want 1
```

## 写足够的代码让测试通过

由于我们只是断言调用次数而不是具体的值，第一次迭代的步子就更小一点。

如果我们想要能调用 `RecordWin`，需要更新 `PlayerServer` 对 `PlayerStore` 的认知，也就是修改这个接口。

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}
```

这样一来 `main` 不再能编译

```
./main.go:17:46: cannot use InMemoryPlayerStore literal (type *InMemoryPlayerStore) as type PlayerStore in field value:
    *InMemoryPlayerStore does not implement PlayerStore (missing RecordWin method)
```

编译器告诉我们哪里出问题了。让我们更新 `InMemoryPlayerStore`，给它加上这个方法。

```go
//main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) RecordWin(name string) {}
```

试着运行测试，我们应该回到能编译的状态了——但测试还是失败的。

既然 `PlayerStore` 现在有了 `RecordWin`，我们就能在 `PlayerServer` 里调用它

```go
//server.go
func (p *PlayerServer) processWin(w http.ResponseWriter) {
	p.store.RecordWin("Bob")
	w.WriteHeader(http.StatusAccepted)
}
```

跑一下测试，应该通过了！显然 `"Bob"` 并不是我们想要传给 `RecordWin` 的，所以让我们继续完善测试。

## 先写测试

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
		nil,
	}
	server := &PlayerServer{&store}

	t.Run("it records wins on POST", func(t *testing.T) {
		player := "Pepper"

		request := newPostWinRequest(player)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)

		if len(store.winCalls) != 1 {
			t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
		}

		if store.winCalls[0] != player {
			t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], player)
		}
	})
}
```

既然我们知道 `winCalls` 切片中有一个元素，就可以安全地引用第一个，并检查它是否等于 `player`。

## 尝试运行测试

```
=== RUN   TestStoreWins/it_records_wins_on_POST
    --- FAIL: TestStoreWins/it_records_wins_on_POST (0.00s)
        server_test.go:86: did not store correct winner got 'Bob' want 'Pepper'
```

## 写足够的代码让测试通过

```go
//server.go
func (p *PlayerServer) processWin(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

我们让 `processWin` 也接收 `http.Request`，这样就能从 URL 中提取玩家名字。拿到名字后，我们就能用正确的值调用 `store`，让测试通过。

## 重构

我们可以把代码 DRY 化一点，因为我们用同样的方式在两处提取玩家名字

```go
//server.go
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

虽然测试通过了，但实际上我们还没真正把软件做好。如果你试着运行 `main` 并按预期使用这个软件，它并不能工作，因为我们还没正确实现 `PlayerStore`。这没问题；通过聚焦 handler，我们识别出了所需的接口，而不是一上来就设计好接口。

我们 _可以_ 给 `InMemoryPlayerStore` 写一些测试，但它只是临时用一下，等我们实现一种更健壮的玩家分数持久化方式（比如数据库）就会替换。

我们现在要做的是写一个 `PlayerServer` 与 `InMemoryPlayerStore` 之间的 _集成测试_ 来收尾这部分功能。这能让我们达成"我对应用是否能工作有信心"的目标，而不必直接测试 `InMemoryPlayerStore`。不仅如此，等我们用数据库实现 `PlayerStore` 时，我们可以用同样的集成测试来测试那个实现。

### 集成测试

集成测试对于测试系统较大的区域是否工作很有用，但你必须记住：

-   它们更难写
-   失败时，可能很难判断原因（通常是集成测试中某个组件里的 bug），因此修起来也更难
-   它们运行有时较慢（因为常常会用到"真实"组件，比如数据库）

正因如此，建议你了解一下 _测试金字塔_。

## 先写测试

为了简洁起见，我直接给你看最终重构后的集成测试。

```go
// server_integration_test.go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := InMemoryPlayerStore{}
	server := PlayerServer{&store}
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	response := httptest.NewRecorder()
	server.ServeHTTP(response, newGetScoreRequest(player))
	assertStatus(t, response.Code, http.StatusOK)

	assertResponseBody(t, response.Body.String(), "3")
}
```

-   我们正在创建准备集成的两个组件：`InMemoryPlayerStore` 和 `PlayerServer`。
-   然后我们发出 3 次请求来记录 `player` 的 3 场胜利。在这个测试里我们不太关心状态码，因为它和它们是否良好集成无关。
-   下一个响应我们才在意（所以保存到变量 `response` 中），因为我们将尝试获取 `player` 的分数。

## 尝试运行测试

```
--- FAIL: TestRecordingWinsAndRetrievingThem (0.00s)
    server_integration_test.go:24: response body is wrong, got '123' want '3'
```

## 写足够的代码让测试通过

我在这里要稍微放宽一点，写比你可能习惯的、不写测试就直接写出来的更多代码。

_这是允许的！_ 我们仍然有测试在确保东西能正确工作，只是不在我们正在工作的具体单元（`InMemoryPlayerStore`）周围。

如果在这种情况下我卡住了，我会回滚改动到失败的测试，然后围绕 `InMemoryPlayerStore` 写更具体的单元测试，帮助我推导出方案。

```go
//in_memory_player_store.go
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

-   我们需要存数据，所以给 `InMemoryPlayerStore` struct 加了一个 `map[string]int`
-   为了方便，我做了 `NewInMemoryPlayerStore` 来初始化 store，并更新集成测试使用它：
    ```go
    //server_integration_test.go
    store := NewInMemoryPlayerStore()
    server := PlayerServer{store}
    ```
-   其余的代码只是把 `map` 包了一层

集成测试通过了，现在我们只需要修改 `main` 来使用 `NewInMemoryPlayerStore()`

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

构建、运行，然后用 `curl` 来测试一下。

-   多运行几次，玩家名字也可以改 `curl -X POST http://localhost:5000/players/Pepper`
-   用 `curl http://localhost:5000/players/Pepper` 查看分数

很好！你做出了一个准 REST 风格的服务。要让这个东西更进一步，你会想选一种数据存储，把分数持久化到比程序运行时间更长的地方。

-   选一种存储（Bolt？Mongo？Postgres？文件系统？）
-   让 `PostgresPlayerStore` 实现 `PlayerStore`
-   用 TDD 实现功能，确保它能工作
-   把它接入集成测试，检查是否仍然通过
-   最后接入 `main`

## 重构

我们快做完了！让我们花点功夫预防这种并发错误

```
fatal error: concurrent map read and map write
```

通过加 mutex，我们确保了并发安全，特别是 `RecordWin` 函数中的计数器。关于 mutex 的更多内容请看 sync 那一章。

## 总结

### `http.Handler`

-   实现这个接口来创建 web 服务器
-   使用 `http.HandlerFunc` 把普通函数转成 `http.Handler`
-   使用 `httptest.NewRecorder` 作为 `ResponseWriter` 传入，可以让你 spy 你的 handler 发出的响应
-   使用 `http.NewRequest` 来构造你期望系统接收到的请求

### 接口、Mock 与 DI

-   让你能小块小块地迭代搭建系统
-   让你能开发一个需要存储的 handler，而不需要真正的存储
-   用 TDD 推导出你需要的接口

### 先犯"罪"，然后重构（再提交到版本控制）

-   你需要把"编译失败或测试失败"当成红色状态，必须尽快脱离它。
-   只写让你脱离这个状态所必需的代码。_然后_ 再重构，把代码做漂亮。
-   在代码不能编译或测试失败时尝试做太多改动，会让你冒着把问题不断叠加的风险。
-   坚持这种方式会迫使你写小测试，意味着小改动，从而让在复杂系统上工作变得可控。
