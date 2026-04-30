# IO 与排序

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/io)**

[在上一章](json.md)中，我们继续迭代我们的应用，新增了一个 `/league` 端点。在这个过程中，我们学到了如何处理 JSON、嵌入类型以及路由。

我们的产品负责人对于服务器重启后软件丢失分数这件事有些不满。原因是我们的 store 实现是基于内存的。她也不太高兴我们没有把 `/league` 端点理解成应当按胜场数排序返回玩家！

## 目前的代码

```go
// server.go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strings"
)

// PlayerStore stores score information about players
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}

// Player stores a name with a number of wins
type Player struct {
	Name string
	Wins int
}

// PlayerServer is a HTTP interface for player information
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

const jsonContentType = "application/json"

// NewPlayerServer creates a PlayerServer with routing configured
func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", jsonContentType)
	json.NewEncoder(w).Encode(p.store.GetLeague())
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

func (i *InMemoryPlayerStore) GetLeague() []Player {
	var league []Player
	for name, wins := range i.store {
		league = append(league, Player{name, wins})
	}
	return league
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
	server := NewPlayerServer(NewInMemoryPlayerStore())
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

你可以在本章顶部的链接中找到对应的测试。

## 存储数据

我们可以选择的数据库有几十种，但我们打算采取一种非常简单的方式。我们会把这个应用的数据以 JSON 的形式存到一个文件里。

这样做让数据非常便携，并且实现起来相对简单。

它的扩展性不会特别好，但鉴于这是一个原型，目前来说足够了。如果情况发生变化、不再合适，由于我们用了 `PlayerStore` 这个抽象，把它换成别的东西也很简单。

我们暂时会保留 `InMemoryPlayerStore`，这样在我们开发新 store 时集成测试还能继续通过。等我们确信新的实现足以让集成测试通过，我们再把它换上去，然后删掉 `InMemoryPlayerStore`。

## 先写测试

到现在你应该已经熟悉标准库中用于读取数据（`io.Reader`）、写入数据（`io.Writer`）的接口，以及如何利用标准库在不使用真实文件的情况下测试这些函数。

为了完成这项工作，我们需要实现 `PlayerStore`，因此我们会针对要实现的方法为我们的 store 编写测试。我们先从 `GetLeague` 开始。

```go
//file_system_store_test.go
func TestFileSystemStore(t *testing.T) {

	t.Run("league from a reader", func(t *testing.T) {
		database := strings.NewReader(`[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)

		store := FileSystemPlayerStore{database}

		got := store.GetLeague()

		want := []Player{
			{"Cleo", 10},
			{"Chris", 33},
		}

		assertLeague(t, got, want)
	})
}
```

我们使用 `strings.NewReader`，它会返回一个 `Reader`，这正是我们的 `FileSystemPlayerStore` 用来读取数据的东西。在 `main` 中我们会打开一个文件，文件也是一个 `Reader`。

## 试着运行测试

```
# github.com/quii/learn-go-with-tests/io/v1
./file_system_store_test.go:15:12: undefined: FileSystemPlayerStore
```

## 写最少量的代码让测试运行起来，并检查失败的测试输出

我们在新文件里定义 `FileSystemPlayerStore`

```go
//file_system_store.go
type FileSystemPlayerStore struct{}
```

再试一次

```
# github.com/quii/learn-go-with-tests/io/v1
./file_system_store_test.go:15:28: too many values in struct initializer
./file_system_store_test.go:17:15: store.GetLeague undefined (type FileSystemPlayerStore has no field or method GetLeague)
```

它在抱怨，因为我们传入了一个 `Reader` 但它没有期待这个参数，而且它还没定义 `GetLeague`。

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.Reader
}

func (f *FileSystemPlayerStore) GetLeague() []Player {
	return nil
}
```

再试一次……

```
=== RUN   TestFileSystemStore//league_from_a_reader
    --- FAIL: TestFileSystemStore//league_from_a_reader (0.00s)
        file_system_store_test.go:24: got [] want [{Cleo 10} {Chris 33}]
```

## 写够让测试通过的代码

我们之前已经从 reader 解析过 JSON

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() []Player {
	var league []Player
	json.NewDecoder(f.database).Decode(&league)
	return league
}
```

测试应该通过了。

## 重构

我们 _之前_ 已经做过这件事了！我们 server 的测试代码也得从响应里解码 JSON。

我们试着把它 DRY 成一个函数。

新建一个文件 `league.go`，把下面的代码放进去。

```go
//league.go
func NewLeague(rdr io.Reader) ([]Player, error) {
	var league []Player
	err := json.NewDecoder(rdr).Decode(&league)
	if err != nil {
		err = fmt.Errorf("problem parsing league, %v", err)
	}

	return league, err
}
```

在我们的实现里调用它，并在 `server_test.go` 的辅助函数 `getLeagueFromResponse` 里也调用它

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() []Player {
	league, _ := NewLeague(f.database)
	return league
}
```

我们还没有处理解析错误的策略，但先继续推进。

### Seek 的问题

我们的实现存在一个缺陷。首先，我们回想一下 `io.Reader` 是怎么定义的。

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

对于我们的文件，你可以想象它一个字节一个字节地读到末尾。如果你尝试再 `Read` 一次会发生什么？

把下面的代码加到我们当前测试的末尾。

```go
//file_system_store_test.go

// read again
got = store.GetLeague()
assertLeague(t, got, want)
```

我们希望这个能通过，但你跑测试的话会发现它通不过。

问题在于我们的 `Reader` 已经读到了末尾，所以没有更多内容可读了。我们需要一种方式告诉它回到开头。

[ReadSeeker](https://golang.org/pkg/io/#ReadSeeker) 是标准库里另一个能帮上忙的接口。

```go
type ReadSeeker interface {
	Reader
	Seeker
}
```

还记得嵌入吗？这个接口由 `Reader` 和 [`Seeker`](https://golang.org/pkg/io/#Seeker) 组合而成

```go
type Seeker interface {
	Seek(offset int64, whence int) (int64, error)
}
```

听起来不错，我们能改 `FileSystemPlayerStore` 让它接受这个接口吗？

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadSeeker
}

func (f *FileSystemPlayerStore) GetLeague() []Player {
	f.database.Seek(0, io.SeekStart)
	league, _ := NewLeague(f.database)
	return league
}
```

试着运行测试，现在通过了！幸运的是我们在测试中用的 `strings.NewReader` 也实现了 `ReadSeeker`，所以我们不需要做其他改动。

接下来我们将实现 `GetPlayerScore`。

## 先写测试

```go
//file_system_store_test.go
t.Run("get player score", func(t *testing.T) {
	database := strings.NewReader(`[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)

	store := FileSystemPlayerStore{database}

	got := store.GetPlayerScore("Chris")

	want := 33

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
})
```

## 试着运行测试

```
./file_system_store_test.go:38:15: store.GetPlayerScore undefined (type FileSystemPlayerStore has no field or method GetPlayerScore)
```

## 写最少量的代码让测试运行起来，并检查失败的测试输出

我们需要给新类型加上这个方法，让测试能编译。

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {
	return 0
}
```

现在它能编译了，测试失败

```
=== RUN   TestFileSystemStore/get_player_score
    --- FAIL: TestFileSystemStore//get_player_score (0.00s)
        file_system_store_test.go:43: got 0 want 33
```

## 写够让测试通过的代码

我们可以遍历 league 找到那位玩家，并返回他的分数

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	var wins int

	for _, player := range f.GetLeague() {
		if player.Name == name {
			wins = player.Wins
			break
		}
	}

	return wins
}
```

## 重构

你已经见过几十次测试辅助函数的重构了，所以这次留给你自己来动手

```go
//file_system_store_test.go
t.Run("get player score", func(t *testing.T) {
	database := strings.NewReader(`[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)

	store := FileSystemPlayerStore{database}

	got := store.GetPlayerScore("Chris")
	want := 33
	assertScoreEquals(t, got, want)
})
```

最后，我们需要开始用 `RecordWin` 来记录分数。

## 先写测试

我们对写入的处理方式相当短视。我们没法（轻易地）只更新文件中 JSON 的某一"行"。我们每次写入都需要把数据库的 _整个_ 新表示存进去。

我们怎么写？通常我们会用 `Writer`，但我们已经有了 `ReadSeeker`。我们其实可以有两个依赖，但标准库已经为我们提供了一个接口 `ReadWriteSeeker`，它能让我们做完所有处理文件需要的事。

我们更新一下类型

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadWriteSeeker
}
```

看看它能不能编译

```
./file_system_store_test.go:15:34: cannot use database (type *strings.Reader) as type io.ReadWriteSeeker in field value:
    *strings.Reader does not implement io.ReadWriteSeeker (missing Write method)
./file_system_store_test.go:36:34: cannot use database (type *strings.Reader) as type io.ReadWriteSeeker in field value:
    *strings.Reader does not implement io.ReadWriteSeeker (missing Write method)
```

`strings.Reader` 没有实现 `ReadWriteSeeker`，这并不太意外，那我们怎么办？

我们有两个选择

- 为每个测试创建一个临时文件。`*os.File` 实现了 `ReadWriteSeeker`。它的好处是这更像一种集成测试，我们真的在读写文件系统，所以会带来非常高的信心。坏处是我们更偏好单元测试，因为它们更快、通常也更简单。我们还需要做更多的工作来创建临时文件，并确保测试结束后把它们清掉。
- 我们可以使用第三方库。[Mattetti](https://github.com/mattetti) 写了一个库 [filebuffer](https://github.com/mattetti/filebuffer)，它实现了我们需要的接口，并且不会触碰文件系统。

我觉得这里没什么特别错的答案，但如果选择用第三方库，我就得去解释依赖管理！所以我们改用文件。

在加我们的测试之前，我们需要把 `strings.Reader` 替换为 `os.File` 让其它测试能编译。

我们来创建一些辅助函数，它们会创建一个内含一些数据的临时文件，并把分数测试抽象出来

```go
//file_system_store_test.go
func createTempFile(t testing.TB, initialData string) (io.ReadWriteSeeker, func()) {
	t.Helper()

	tmpfile, err := os.CreateTemp("", "db")

	if err != nil {
		t.Fatalf("could not create temp file %v", err)
	}

	tmpfile.Write([]byte(initialData))

	removeFile := func() {
		tmpfile.Close()
		os.Remove(tmpfile.Name())
	}

	return tmpfile, removeFile
}

func assertScoreEquals(t testing.TB, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

[CreateTemp](https://pkg.go.dev/os#CreateTemp) 为我们创建一个临时文件以供使用。我们传入的 `"db"` 是它创建的随机文件名上的前缀。这是为了确保它不会意外地与其它文件冲突。

你会注意到我们不仅返回了 `ReadWriteSeeker`（也就是文件），还返回了一个函数。我们要确保测试结束后文件被删除。我们不希望把文件的细节泄漏到测试里，因为这容易出错，对读者来说也没意思。通过返回一个 `removeFile` 函数，我们可以在辅助函数里处理这些细节，调用方只需要执行 `defer cleanDatabase()`。

```go
//file_system_store_test.go
func TestFileSystemStore(t *testing.T) {

	t.Run("league from a reader", func(t *testing.T) {
		database, cleanDatabase := createTempFile(t, `[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)
		defer cleanDatabase()

		store := FileSystemPlayerStore{database}

		got := store.GetLeague()

		want := []Player{
			{"Cleo", 10},
			{"Chris", 33},
		}

		assertLeague(t, got, want)

		// read again
		got = store.GetLeague()
		assertLeague(t, got, want)
	})

	t.Run("get player score", func(t *testing.T) {
		database, cleanDatabase := createTempFile(t, `[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)
		defer cleanDatabase()

		store := FileSystemPlayerStore{database}

		got := store.GetPlayerScore("Chris")
		want := 33
		assertScoreEquals(t, got, want)
	})
}
```

运行测试，它们应该都能通过！改动相当多，但现在感觉我们的接口定义已经完整，从此添加新测试应该会很容易。

我们来做记录已存在玩家胜场的第一版迭代

```go
//file_system_store_test.go
t.Run("store wins for existing players", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer cleanDatabase()

	store := FileSystemPlayerStore{database}

	store.RecordWin("Chris")

	got := store.GetPlayerScore("Chris")
	want := 34
	assertScoreEquals(t, got, want)
})
```

## 试着运行测试

`./file_system_store_test.go:67:8: store.RecordWin undefined (type FileSystemPlayerStore has no field or method RecordWin)`

## 写最少量的代码让测试运行起来，并检查失败的测试输出

加上新方法

```go
//file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {

}
```

```
=== RUN   TestFileSystemStore/store_wins_for_existing_players
    --- FAIL: TestFileSystemStore/store_wins_for_existing_players (0.00s)
        file_system_store_test.go:71: got 33 want 34
```

我们的实现是空的，所以返回的是旧分数。

## 写够让测试通过的代码

```go
//file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()

	for i, player := range league {
		if player.Name == name {
			league[i].Wins++
		}
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

你可能会问，为什么我写的是 `league[i].Wins++` 而不是 `player.Wins++`。

当你 `range` 遍历一个切片时，会得到当前循环的索引（这里是 `i`）以及该索引上元素的一份 _副本_。修改副本的 `Wins` 值不会对我们正在遍历的 `league` 切片产生任何影响。因此，我们需要通过 `league[i]` 拿到真正值的引用，再去改它。

如果你运行测试，它们现在应该都能通过了。

## 重构

在 `GetPlayerScore` 和 `RecordWin` 中，我们都遍历 `[]Player` 来按名字查找玩家。

我们可以把这段公共代码重构到 `FileSystemStore` 内部，但在我看来，这段代码也许有用，可以提取成一个新类型。到目前为止，我们处理"League"用的一直是 `[]Player`，但我们可以创建一个新类型叫 `League`。这对其他开发者来说会更容易理解，然后我们可以在那个类型上挂一些有用的方法供我们使用。

在 `league.go` 里加上下面的代码

```go
//league.go
type League []Player

func (l League) Find(name string) *Player {
	for i, p := range l {
		if p.Name == name {
			return &l[i]
		}
	}
	return nil
}
```

现在如果谁手上有一个 `League`，他就可以轻松地查找指定玩家了。

把我们的 `PlayerStore` 接口改成返回 `League` 而不是 `[]Player`。再次运行测试，你会得到一个编译错误，因为我们改动了接口，但很容易修；只要把返回类型从 `[]Player` 改成 `League` 即可。

这让我们能简化 `file_system_store` 中的方法。

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	player := f.GetLeague().Find(name)

	if player != nil {
		return player.Wins
	}

	return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()
	player := league.Find(name)

	if player != nil {
		player.Wins++
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

这看上去好多了，而且我们能想到围绕 `League` 还有其它一些有用功能可以重构出来。

我们现在需要处理记录新玩家胜场的场景。

## 先写测试

```go
//file_system_store_test.go
t.Run("store wins for new players", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer cleanDatabase()

	store := FileSystemPlayerStore{database}

	store.RecordWin("Pepper")

	got := store.GetPlayerScore("Pepper")
	want := 1
	assertScoreEquals(t, got, want)
})
```

## 试着运行测试

```
=== RUN   TestFileSystemStore/store_wins_for_new_players#01
    --- FAIL: TestFileSystemStore/store_wins_for_new_players#01 (0.00s)
        file_system_store_test.go:86: got 0 want 1
```

## 写够让测试通过的代码

我们只需要处理 `Find` 因为找不到玩家而返回 `nil` 的场景。

```go
//file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()
	player := league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		league = append(league, Player{name, 1})
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

正常路径看起来没问题，所以现在我们可以在集成测试中尝试使用我们的新 `Store`。这能给我们更多信心相信软件是好用的，然后我们就可以删掉多余的 `InMemoryPlayerStore`。

在 `TestRecordingWinsAndRetrievingThem` 里替换掉旧的 store。

```go
//server_integration_test.go
database, cleanDatabase := createTempFile(t, "")
defer cleanDatabase()
store := &FileSystemPlayerStore{database}
```

如果你运行测试它应该通过，现在我们可以删掉 `InMemoryPlayerStore` 了。`main.go` 现在会有编译问题，这会促使我们在"真实"代码里使用我们的新 store。

```go
// main.go
package main

import (
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

	store := &FileSystemPlayerStore{db}
	server := NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

- 我们为数据库创建一个文件。
- `os.OpenFile` 的第 2 个参数让你定义打开文件时的权限，在我们的例子中 `O_RDWR` 表示我们想读写，`os.O_CREATE` 则表示文件不存在时创建。
- 第 3 个参数表示设置文件的权限，在我们的例子中，所有用户都可以读写该文件。[（详细解释见 superuser.com）](https://superuser.com/questions/295591/what-is-the-meaning-of-chmod-666)。

现在运行程序，数据会在重启之间持久化到文件里，万岁！

## 进一步的重构与性能考量

每次有人调用 `GetLeague()` 或 `GetPlayerScore()` 时，我们都在读取整个文件并解析成 JSON。我们其实没必要这样做，因为 `FileSystemStore` 完全负责 league 的状态；它只需要在程序启动时读取一次文件，并在数据变化时更新文件。

我们可以创建一个构造函数为我们做一些初始化工作，并把 league 作为值存在 `FileSystemStore` 上，让读取时使用这个值。

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadWriteSeeker
	league   League
}

func NewFileSystemPlayerStore(database io.ReadWriteSeeker) *FileSystemPlayerStore {
	database.Seek(0, io.SeekStart)
	league, _ := NewLeague(database)
	return &FileSystemPlayerStore{
		database: database,
		league:   league,
	}
}
```

这样我们只需要从磁盘读一次。我们现在可以把之前所有从磁盘获取 league 的调用都替换为使用 `f.league`。

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() League {
	return f.league
}

func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	player := f.league.Find(name)

	if player != nil {
		return player.Wins
	}

	return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
	player := f.league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		f.league = append(f.league, Player{name, 1})
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(f.league)
}
```

如果你运行测试，它会抱怨 `FileSystemPlayerStore` 的初始化方式有问题，那就把它们改为调用我们的新构造函数即可。

### 另一个问题

我们处理文件的方式还有些天真，_可能_ 在以后埋下一个非常讨厌的 bug。

当我们 `RecordWin` 时，我们 `Seek` 回文件开头然后写入新数据——但如果新数据比之前的数据小怎么办？

在我们目前的场景里这不可能发生。我们永远不会编辑或删除分数，所以数据只会变大。然而把代码这样留下也是不负责任的；删除场景出现并非不可想象。

那我们要怎么测试这个呢？我们要做的是先重构代码，把 _我们写什么样的数据_ 与 _写入操作本身_ 这两个关注点分离开。然后我们可以单独测试它，看它能不能按我们希望的方式工作。

我们会创建一个新类型来封装我们这种"写入时从开头开始"的功能。我打算把它叫做 `Tape`。新建一个文件，写入下面的内容：

```go
// tape.go
package main

import "io"

type tape struct {
	file io.ReadWriteSeeker
}

func (t *tape) Write(p []byte) (n int, err error) {
	t.file.Seek(0, io.SeekStart)
	return t.file.Write(p)
}
```

注意我们现在只实现 `Write`，因为它封装了 `Seek` 那部分。这意味着我们的 `FileSystemStore` 只需要持有一个 `Writer` 的引用即可。

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.Writer
	league   League
}
```

更新构造函数以使用 `Tape`

```go
//file_system_store.go
func NewFileSystemPlayerStore(database io.ReadWriteSeeker) *FileSystemPlayerStore {
	database.Seek(0, io.SeekStart)
	league, _ := NewLeague(database)

	return &FileSystemPlayerStore{
		database: &tape{database},
		league:   league,
	}
}
```

最后，我们可以从 `RecordWin` 里移除 `Seek` 调用，得到我们想要的酷炫回报。是的，这看起来不算什么，但至少这意味着如果我们做其它种类的写入，我们可以依赖我们的 `Write` 按需要的方式工作。而且现在我们可以单独测试这段潜在有问题的代码，并修复它。

我们来写一个测试，希望用比原内容更小的内容更新整个文件。

## 先写测试

我们的测试会创建一个含有内容的文件，尝试用 `tape` 写入，然后再读出来看文件里有什么。在 `tape_test.go` 里：

```go
//tape_test.go
func TestTape_Write(t *testing.T) {
	file, clean := createTempFile(t, "12345")
	defer clean()

	tape := &tape{file}

	tape.Write([]byte("abc"))

	file.Seek(0, io.SeekStart)
	newFileContents, _ := io.ReadAll(file)

	got := string(newFileContents)
	want := "abc"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

## 试着运行测试

```
=== RUN   TestTape_Write
--- FAIL: TestTape_Write (0.00s)
    tape_test.go:23: got 'abc45' want 'abc'
```

正如我们所想！它写入了我们想要的数据，但留下了原数据剩下的部分。

## 写够让测试通过的代码

`os.File` 有一个 truncate 函数可以让我们有效地清空文件。我们应该可以直接调用它来达到我们的目的。

把 `tape` 改成下面这样：

```go
//tape.go
type tape struct {
	file *os.File
}

func (t *tape) Write(p []byte) (n int, err error) {
	t.file.Truncate(0)
	t.file.Seek(0, io.SeekStart)
	return t.file.Write(p)
}
```

编译器会在很多地方失败，因为我们期待的是 `io.ReadWriteSeeker`，但传入的是 `*os.File`。到现在你应该自己就能修这些问题了，如果卡住了就看看源码。

修好后，我们的 `TestTape_Write` 测试就应该能通过了！

### 再做一个小重构

在 `RecordWin` 里我们有这一行 `json.NewEncoder(f.database).Encode(f.league)`。

我们没必要每次写入时都创建一个新的 encoder，我们可以在构造函数里初始化一个并复用它。

在我们的类型里存一个 `Encoder` 的引用，并在构造函数里初始化它：

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database *json.Encoder
	league   League
}

func NewFileSystemPlayerStore(file *os.File) *FileSystemPlayerStore {
	file.Seek(0, io.SeekStart)
	league, _ := NewLeague(file)

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}
}
```

在 `RecordWin` 中使用它。

```go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	player := f.league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		f.league = append(f.league, Player{name, 1})
	}

	f.database.Encode(f.league)
}
```

## 我们刚才不是违反了一些规则吗？测试私有的东西？没有接口？

### 关于测试私有类型

确实，_一般来说_ 你应该倾向于不去测试私有的东西，因为这有时会导致测试与实现耦合得太紧，将来阻碍重构。

但是我们不能忘记，测试应当给我们带来 _信心_。

我们之前不确信加了任何编辑或删除功能后我们的实现还能正常工作。我们不想把代码就这样留着，尤其是当这是由不止一个人维护时，他们可能并不知晓我们最初做法的不足之处。

最后，这只是一个测试！如果将来我们决定改变它的工作方式，删掉它也没什么大不了的，但至少我们已经为未来的维护者捕捉到了这个需求。

### 接口

我们一开始用的是 `io.Reader`，因为这是我们最容易对新 `PlayerStore` 做单元测试的路径。在开发过程中我们换成了 `io.ReadWriter`，再换成 `io.ReadWriteSeeker`。然后我们发现标准库里实际上除了 `*os.File` 没有别的东西实现这个接口。我们本来可以决定自己写一个或用一个开源的，但务实地看用临时文件做测试就好。

最终，我们还需要 `Truncate`，它也在 `*os.File` 上。我们本来也可以选择创建自己的接口来囊括这些需求。

```go
type ReadWriteSeekTruncate interface {
	io.ReadWriteSeeker
	Truncate(size int64) error
}
```

但这真的能给我们带来什么？请记住我们 _并没有 mock_，而对于一个 **文件系统** store 来说，接受除 `*os.File` 之外的任何类型也是不现实的，所以我们不需要接口提供的多态性。

不要害怕像我们这里这样切换、改变类型并做实验。使用静态类型语言的好处就是，每次改动编译器都会帮你。

## 错误处理

在我们开始处理排序之前，应该确认对当前的代码满意，并消除可能存在的技术债。尽快把软件做到能工作（不要陷在红色状态）是一项重要原则，但这并不意味着我们应该忽略错误情况！

回到 `FileSystemStore.go`，我们的构造函数里有 `league, _ := NewLeague(f.database)`。

`NewLeague` 在无法从我们提供的 `io.Reader` 解析 league 时会返回一个错误。

当时忽略它是务实的，因为我们已经有正在失败的测试。如果当时同时去处理它，我们就要同时兼顾两件事。

我们让构造函数能够返回错误。

```go
//file_system_store.go
func NewFileSystemPlayerStore(file *os.File) (*FileSystemPlayerStore, error) {
	file.Seek(0, io.SeekStart)
	league, err := NewLeague(file)

	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %s, %v", file.Name(), err)
	}

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}, nil
}
```

记住，提供有用的错误信息非常重要（就像你的测试一样）。网上有人开玩笑说大多数 Go 代码就是：

```go
if err != nil {
	return err
}
```

**这 100% 不是地道的写法。** 给错误信息添加上下文（也就是你做了什么导致了这个错误）能让运维你的软件容易得多。

如果你尝试编译，会得到一些错误。

```
./main.go:18:35: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:35:36: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:57:36: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:70:36: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:85:36: multiple-value NewFileSystemPlayerStore() in single-value context
./server_integration_test.go:12:35: multiple-value NewFileSystemPlayerStore() in single-value context
```

在 main 里我们想让程序退出，并打印错误。

```go
//main.go
store, err := NewFileSystemPlayerStore(db)

if err != nil {
	log.Fatalf("problem creating file system player store, %v ", err)
}
```

在测试里我们应该断言没有错误。我们可以做一个辅助函数来帮忙。

```go
//file_system_store_test.go
func assertNoError(t testing.TB, err error) {
	t.Helper()
	if err != nil {
		t.Fatalf("didn't expect an error but got one, %v", err)
	}
}
```

利用这个辅助函数解决其它编译问题。最后，你应该会有一个失败的测试：

```
=== RUN   TestRecordingWinsAndRetrievingThem
--- FAIL: TestRecordingWinsAndRetrievingThem (0.00s)
    server_integration_test.go:14: didn't expect an error but got one, problem loading player store from file /var/folders/nj/r_ccbj5d7flds0sf63yy4vb80000gn/T/db841037437, problem parsing league, EOF
```

我们没法解析 league，因为文件是空的。之前没报错是因为我们一直忽略了错误。

我们把那个大的集成测试修一下，往里面放一些有效的 JSON：

```go
//server_integration_test.go
func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[]`)
	//etc...
}
```

现在所有测试都通过了，我们需要处理文件为空的场景。

## 先写测试

```go
//file_system_store_test.go
t.Run("works with an empty file", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, "")
	defer cleanDatabase()

	_, err := NewFileSystemPlayerStore(database)

	assertNoError(t, err)
})
```

## 试着运行测试

```
=== RUN   TestFileSystemStore/works_with_an_empty_file
    --- FAIL: TestFileSystemStore/works_with_an_empty_file (0.00s)
        file_system_store_test.go:108: didn't expect an error but got one, problem loading player store from file /var/folders/nj/r_ccbj5d7flds0sf63yy4vb80000gn/T/db019548018, problem parsing league, EOF
```

## 写够让测试通过的代码

把构造函数改成下面这样

```go
//file_system_store.go
func NewFileSystemPlayerStore(file *os.File) (*FileSystemPlayerStore, error) {

	file.Seek(0, io.SeekStart)

	info, err := file.Stat()

	if err != nil {
		return nil, fmt.Errorf("problem getting file info from file %s, %v", file.Name(), err)
	}

	if info.Size() == 0 {
		file.Write([]byte("[]"))
		file.Seek(0, io.SeekStart)
	}

	league, err := NewLeague(file)

	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %s, %v", file.Name(), err)
	}

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}, nil
}
```

`file.Stat` 返回我们文件的统计信息，让我们能检查文件大小。如果是空的，我们就 `Write` 一个空的 JSON 数组并 `Seek` 回开头，为后续代码做好准备。

## 重构

我们的构造函数现在有点乱，所以我们把初始化代码抽到一个函数里：

```go
//file_system_store.go
func initialisePlayerDBFile(file *os.File) error {
	file.Seek(0, io.SeekStart)

	info, err := file.Stat()

	if err != nil {
		return fmt.Errorf("problem getting file info from file %s, %v", file.Name(), err)
	}

	if info.Size() == 0 {
		file.Write([]byte("[]"))
		file.Seek(0, io.SeekStart)
	}

	return nil
}
```

```go
//file_system_store.go
func NewFileSystemPlayerStore(file *os.File) (*FileSystemPlayerStore, error) {

	err := initialisePlayerDBFile(file)

	if err != nil {
		return nil, fmt.Errorf("problem initialising player db file, %v", err)
	}

	league, err := NewLeague(file)

	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %s, %v", file.Name(), err)
	}

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}, nil
}
```

## 排序

我们的产品负责人希望 `/league` 按分数从高到低排序返回玩家。

这里要做的主要决定是这件事应该在软件的哪个层面发生。如果我们用的是"真正的"数据库，我们会使用类似 `ORDER BY` 这样的东西，排序会非常快。出于这个原因，感觉应该由 `PlayerStore` 的实现来负责。

## 先写测试

我们可以更新 `TestFileSystemStore` 中第一个测试的断言：

```go
//file_system_store_test.go
t.Run("league sorted", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer cleanDatabase()

	store, err := NewFileSystemPlayerStore(database)

	assertNoError(t, err)

	got := store.GetLeague()

	want := League{
		{"Chris", 33},
		{"Cleo", 10},
	}

	assertLeague(t, got, want)

	// read again
	got = store.GetLeague()
	assertLeague(t, got, want)
})
```

进入的 JSON 顺序是错的，而我们的 `want` 会检查它以正确的顺序返回给调用方。

## 试着运行测试

```
=== RUN   TestFileSystemStore/league_from_a_reader,_sorted
    --- FAIL: TestFileSystemStore/league_from_a_reader,_sorted (0.00s)
        file_system_store_test.go:46: got [{Cleo 10} {Chris 33}] want [{Chris 33} {Cleo 10}]
        file_system_store_test.go:51: got [{Cleo 10} {Chris 33}] want [{Chris 33} {Cleo 10}]
```

## 写够让测试通过的代码

```go
func (f *FileSystemPlayerStore) GetLeague() League {
	sort.Slice(f.league, func(i, j int) bool {
		return f.league[i].Wins > f.league[j].Wins
	})
	return f.league
}
```

[`sort.Slice`](https://golang.org/pkg/sort/#Slice)

> Slice sorts the provided slice given the provided less function.

简单！

## 总结

### 我们涵盖了什么

- `Seeker` 接口，以及它和 `Reader`、`Writer` 的关系。
- 处理文件。
- 创建一个易用的辅助函数来用文件做测试，把那些杂乱的部分都隐藏起来。
- 用 `sort.Slice` 来对切片排序。
- 利用编译器帮助我们安全地对应用做结构性改动。

### 打破规则

- 软件工程里的大多数规则其实并不是规则，只是 80% 时间适用的最佳实践。
- 我们发现一个场景，其中之前的"规则"——不测试内部函数——并不适合我们，所以我们打破了它。
- 在打破规则时，理解你做出的取舍很重要。在我们的场景里，我们是 OK 的，因为这只是一个测试，而且不这样做就很难触发那个场景。
- 想要打破规则，**你必须先理解规则**。一个类比是学吉他。无论你觉得自己多么有创意，你必须理解并练习基本功。

### 我们的软件目前到了哪里

- 我们有了一个 HTTP API，你可以通过它创建玩家并增加他们的分数。
- 我们能以 JSON 形式返回所有人的分数 league。
- 数据以 JSON 文件持久化。
