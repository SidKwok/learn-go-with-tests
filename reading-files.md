# 读取文件

* [**本章的所有代码可以在这里找到**](https://github.com/quii/learn-go-with-tests/tree/main/reading-files)
* [这是我处理这个问题并在 Twitch 直播中回答提问的视频](https://www.youtube.com/watch?v=nXts4dEJnkU)

在本章中，我们将学习如何读取一些文件、从中提取数据，并做一些有用的事情。

设想你在和朋友一起开发一个博客软件。我们的想法是：作者用 markdown 写他们的文章，文件顶部有一些元数据。在启动时，Web 服务器会读取一个文件夹来创建一些 `Post`，然后由一个独立的 `NewHandler` 函数把这些 `Post` 当作博客 Web 服务的数据源使用。

我们被要求开发一个包，把给定的博客文章文件夹转换成 `Post` 的集合。

### 示例数据

hello world.md

```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

The body of posts starts after the `---`
```

### 期望的数据

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

## 迭代式的、测试驱动的开发

我们会采用一种迭代的方式，始终朝着目标走简单、安全的小步。

这要求我们把工作拆分开来，但我们应当小心，不要落入["自下而上"](https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design)方式的陷阱。

我们在开始工作时不应当过于相信自己活跃的想象力。我们可能会忍不住做一些抽象，比如某种 `BlogPostFileParser`，但这种抽象的合理性只有在所有东西拼到一起后才能验证。

这 _不是_ 迭代，而且错过了 TDD 本应给我们带来的紧密反馈循环。

Kent Beck 说：

> 乐观是编程的一种职业病。反馈是它的解药。

相反，我们的方法应该尽快交付 _真实的_ 用户价值（通常被称为"happy path"）。一旦我们端到端地交付了一小块用户价值，剩余需求的迭代往往就会简单直接。

## 思考我们想看到的测试是什么样

让我们提醒自己开始时的心态和目标：

* **写出我们想看到的测试**。从使用者的角度思考我们将要写的代码该如何被使用。
* 关注 _做什么_ 和 _为什么_，但不要被 _怎么做_ 分心。

我们的包需要提供一个函数，可以指向一个文件夹，并返回一些文章。

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS("some-folder")
```

为了围绕它写测试，我们需要某种带有示例文章的测试文件夹。_这样做并没有什么大问题_，但你做了一些权衡：

* 对于每个测试，你可能需要新建文件来测试某种特定行为
* 一些行为会很难测试，例如加载文件失败
* 测试运行得会稍慢一些，因为它们需要访问文件系统

我们也在不必要地把自己耦合到了文件系统的某个具体实现上。

### Go 1.16 引入的文件系统抽象

Go 1.16 为文件系统引入了一个抽象：[io/fs](https://golang.org/pkg/io/fs/) 包。

> fs 包定义了文件系统的基本接口。文件系统可以由宿主操作系统提供，也可以由其他包提供。

这让我们可以放松对具体文件系统的耦合，让我们可以根据需要注入不同的实现。

> [在接口的生产者一侧，新的 embed.FS 类型实现了 fs.FS，zip.Reader 也是。新的 os.DirFS 函数则提供了一个由操作系统文件树支撑的 fs.FS 实现。](https://golang.org/doc/go1.16#fs)

如果我们使用这个接口，我们包的使用者就有一些标准库内置的选项可以选用。学会利用 Go 标准库中定义的接口（例如 `io.fs`、[`io.Reader`](https://golang.org/pkg/io/#Reader)、[`io.Writer`](https://golang.org/pkg/io/#Writer)），对编写松耦合的包至关重要。这些包能在你最初想象之外的不同上下文中被复用，使用者也几乎不需要任何额外操作。

在我们的案例里，也许使用者希望把博客文章嵌入到 Go 二进制文件中，而不是放在"真实"文件系统的文件里？无论如何，_我们的代码不需要关心这一点_。

对于我们的测试，[testing/fstest](https://golang.org/pkg/testing/fstest/) 包给我们提供了一个 [io/FS](https://golang.org/pkg/io/fs/#FS) 的实现可以使用，类似我们熟悉的 [net/http/httptest](https://golang.org/pkg/net/http/httptest/) 中的工具。

基于这些信息，下面这种方式感觉更好：

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS(someFS)
```

## 先写测试

我们应该让范围尽可能小且有用。如果我们能证明可以读取一个目录里的所有文件，那是一个好的开始。这会让我们对正在编写的软件有信心。我们可以检查返回的 `[]Post` 数量是否与我们假文件系统中的文件数相同。

新建一个项目来跟随本章操作。

* `mkdir blogposts`
* `cd blogposts`
* `go mod init github.com/{your-name}/blogposts`
* `touch blogposts_test.go`

```go
package blogposts_test

import (
	"testing"
	"testing/fstest"
)

func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts := blogposts.NewPostsFromFS(fs)

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}
```

注意我们测试的包是 `blogposts_test`。记住，TDD 实践得当时我们采取一种 _以使用者为驱动_ 的方法：我们不想测试内部细节，因为 _使用者_ 并不关心它们。通过在我们打算用的包名后面加上 `_test`，我们只能访问该包导出的成员——就像该包真正的使用者那样。

我们引入了 [`testing/fstest`](https://golang.org/pkg/testing/fstest/)，它让我们可以使用 [`fstest.MapFS`](https://golang.org/pkg/testing/fstest/#MapFS) 类型。我们的假文件系统会把 `fstest.MapFS` 传递给我们的包。

> MapFS 是一个用于测试的简单内存文件系统，它表示为一个 map：从路径名（传给 Open 的参数）映射到该路径所代表的文件或目录的信息。

这感觉比维护一个测试文件夹简单，也会执行得更快。

最后，我们从使用者的角度把 API 的用法固定了下来，然后检查它是否创建了正确数量的文章。

## 尝试运行测试

```
./blogpost_test.go:15:12: undefined: blogposts
```

## 写最少的代码让测试能跑起来，并 _检查失败的测试输出_

这个包还不存在。新建一个文件 `blogposts.go`，并把 `package blogposts` 放进去。然后你需要在测试中导入这个包。对我来说，导入现在是这样的：

```go
import (
	blogposts "github.com/quii/learn-go-with-tests/reading-files"
	"testing"
	"testing/fstest"
)
```

现在测试会编译失败，因为我们的新包还没有 `NewPostsFromFS` 函数返回某种集合。

```
./blogpost_test.go:16:12: undefined: blogposts.NewPostsFromFS
```

这迫使我们做出函数的骨架来让测试运行。记住此时不要过度设计代码；我们只是想让测试能跑起来，并确保它如我们预期那样失败。如果跳过这一步，可能会跳过一些假设，写出一个没用的测试。

```go
package blogposts

import "testing/fstest"

type Post struct {
}

func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return nil
}
```

测试现在应该能正确地失败：

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:48: got 0 posts, wanted 2 posts
```

## 写足够的代码让它通过

我们 _可以_ ["糊弄"](https://deniseyu.github.io/leveling-up-tdd/)（slime）地让它通过：

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return []Post{{}, {}}
}
```

但正如 Denise Yu 所写：

> Sliming（糊弄实现）有助于先给对象搭出一个"骨架"。设计接口和实现逻辑是两件事，策略性地把测试糊弄过去，能让你一次只专注于其中一件。

我们已经有了结构。那么我们应该怎么做呢？

由于我们已经收窄了范围，我们要做的就只是读取目录，并为遇到的每个文件创建一个 post。我们暂时不需要担心打开文件和解析它们。

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

[`fs.ReadDir`](https://golang.org/pkg/io/fs/#ReadDir) 读取给定 `fs.FS` 中的一个目录，返回 [`[]DirEntry`](https://golang.org/pkg/io/fs/#DirEntry)。

我们对世界的理想化看法已经被打破，因为错误是会发生的。但请记住，我们现在的关注点是 _让测试通过_，而不是改变设计，所以暂时忽略这个错误。

剩下的代码很直接：遍历每一项，为每一项创建一个 `Post`，然后返回切片。

## 重构

虽然测试通过了，但我们这个新包在这个上下文之外还用不上，因为它和具体实现 `fstest.MapFS` 耦合了。但其实没必要这样。把 `NewPostsFromFS` 函数的参数改成接受标准库中的接口。

```go
func NewPostsFromFS(fileSystem fs.FS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

重新运行测试：一切都应该正常工作。

### 错误处理

我们之前在专注让正常路径工作的时候把错误处理搁置了。在继续迭代功能之前，我们应该承认操作文件时错误是会发生的。除了读取目录之外，在打开单个文件时也可能遇到问题。我们来修改 API（自然是先改测试），让它能返回一个 `error`。

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts, err := blogposts.NewPostsFromFS(fs)

	if err != nil {
		t.Fatal(err)
	}

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}
```

运行测试：它应该会抱怨返回值数量不对。修复代码很简单。

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts, nil
}
```

这会让测试通过。你内心的 TDD 实践者可能会被这点惹恼：在写传播 `fs.ReadDir` 错误的代码之前，我们没有看到一个失败的测试。要"正确地"做这件事，我们需要写一个新的测试，注入一个会失败的 `fs.FS` 测试替身（test-double），让 `fs.ReadDir` 返回 `error`。

```go
type StubFailingFS struct {
}

func (s StubFailingFS) Open(name string) (fs.File, error) {
	return nil, errors.New("oh no, i always fail")
}
```

```go
// 后面
_, err := blogposts.NewPostsFromFS(StubFailingFS{})
```

这应该让你对我们的方法有信心。我们使用的接口只有一个方法，使得为不同场景创建测试替身变得轻而易举。

在某些情况下，测试错误处理是务实的做法，但在我们的情况里，我们对错误并没有做什么 _有意思的事_，只是在传播它，所以不值得花精力再写一个新的测试。

逻辑上，我们接下来的迭代会围绕扩展 `Post` 类型，让它包含一些有用的数据。

## 先写测试

我们从博客文章规范的第一行——title 字段——开始。

我们需要修改测试文件的内容，让它符合规范，然后我们就可以断言它被正确解析。

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("Title: Post 1")},
		"hello-world2.md": {Data: []byte("Title: Post 2")},
	}

	// 为简洁起见省略其余测试代码
	got := posts[0]
	want := blogposts.Post{Title: "Post 1"}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

## 尝试运行测试

```
./blogpost_test.go:58:26: unknown field 'Title' in struct literal of type blogposts.Post
```

## 写最少的代码让测试能跑起来，并检查失败的测试输出

把新字段加到我们的 `Post` 类型里，让测试能跑起来

```go
type Post struct {
	Title string
}
```

重新运行测试，你应该会看到一个清晰的失败测试

```
=== RUN   TestNewBlogPosts
=== RUN   TestNewBlogPosts/parses_the_post
    blogpost_test.go:61: got {Title:}, want {Title:Post 1}
```

## 写足够的代码让它通过

我们需要打开每个文件然后提取 title

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f)
		if err != nil {
			return nil, err //todo: needs clarification, should we totally fail if one file fails? or just ignore?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()

	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

记住，我们现在的关注点不是写优雅的代码，而是要先达到可以工作的软件这一步。

虽然这感觉像是向前迈了一小步，但仍然让我们写了不少代码，并对错误处理做了一些假设。这是你应该和同事讨论、决定最佳方案的时间点。

迭代式的方法给了我们快速的反馈，让我们意识到自己对需求的理解还不完整。

`fs.FS` 通过 `Open` 方法让我们可以按名字打开它内部的一个文件。从那里我们读取文件中的数据，目前我们不需要复杂的解析，只是通过切片字符串把 `Title:` 前缀去掉。

## 重构

把"打开文件的代码"和"解析文件内容的代码"分离开来，会让代码更易理解和工作。

```go
func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile fs.File) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

当你重构出新的函数或方法时，要小心并思考参数。你在做设计，可以自由地深入思考什么是合适的，因为你有通过的测试做支撑。要思考耦合和内聚。在这个例子里你应该问自己：

> `newPost` 必须和 `fs.File` 耦合吗？我们用到这个类型的所有方法和数据吗？我们 _真正_ 需要的是什么？

在我们的情况里，我们只把它作为参数传给 `io.ReadAll`，而它需要的是 `io.Reader`。所以我们应该松开函数中的耦合，要求一个 `io.Reader`。

```go
func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

对于 `getPost` 函数你也可以提出类似的论点，它接受一个 `fs.DirEntry` 参数，但只是调用 `Name()` 来获取文件名。我们不需要那么多东西；让我们解耦这个类型，把文件名作为字符串传过去。下面是完全重构后的代码：

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f.Name())
		if err != nil {
			return nil, err //todo: needs clarification, should we totally fail if one file fails? or just ignore?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, fileName string) (Post, error) {
	postFile, err := fileSystem.Open(fileName)
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

从现在起，我们大部分的工作可以整齐地放在 `newPost` 里。打开和遍历文件的事已经做完了，现在我们可以专注于为 `Post` 类型提取数据。虽然技术上没有必要，但文件是把相关事物在逻辑上归到一起的好方式，所以我把 `Post` 类型和 `newPost` 移到了一个新的 `post.go` 文件里。

### 测试辅助函数

我们也应该照顾一下测试。我们会经常对 `Posts` 做断言，所以应该写些代码来辅助这件事

```go
func assertPost(t *testing.T, got blogposts.Post, want blogposts.Post) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

```go
assertPost(t, posts[0], blogposts.Post{Title: "Post 1"})
```

## 先写测试

让我们扩展测试，从文件中提取下一行——description。直到让它通过这一过程现在应该感觉熟悉而舒适了。

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1`
		secondBody = `Title: Post 2
Description: Description 2`
	)

	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte(firstBody)},
		"hello-world2.md": {Data: []byte(secondBody)},
	}

	// 为简洁起见省略其余测试代码
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
	})

}
```

## 尝试运行测试

```
./blogpost_test.go:47:58: unknown field 'Description' in struct literal of type blogposts.Post
```

## 写最少的代码让测试能跑起来，并检查失败的测试输出

把新字段加到 `Post` 中。

```go
type Post struct {
	Title       string
	Description string
}
```

测试现在应该能编译，并失败。

```
=== RUN   TestNewBlogPosts
    blogpost_test.go:47: got {Title:Post 1
        Description: Description 1 Description:}, want {Title:Post 1 Description:Description 1}
```

## 写足够的代码让它通过

标准库有一个很方便的库，可以帮你按行扫描数据：[`bufio.Scanner`](https://golang.org/pkg/bufio/#Scanner)

> Scanner 提供了一个便捷的接口，用来读取诸如以换行分隔的文本文件这样的数据。

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	scanner.Scan()
	titleLine := scanner.Text()

	scanner.Scan()
	descriptionLine := scanner.Text()

	return Post{Title: titleLine[7:], Description: descriptionLine[13:]}, nil
}
```

很方便的是它也接受一个 `io.Reader`（再次感谢松耦合），我们不需要修改函数参数。

调用 `Scan` 读一行，然后用 `Text` 提取数据。

这个函数其实永远不会返回 `error`。此时你可能会想把它从返回类型中去掉，但我们知道之后还要处理无效的文件结构，所以不如先留着。

## 重构

我们围绕"扫描一行然后读取文本"有重复。我们知道这个操作至少还会再用一次，DRY 起来是个简单的重构，让我们从这里开始。

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[7:]
	description := readLine()[13:]

	return Post{Title: title, Description: description}, nil
}
```

这几乎没省下几行代码，但这通常不是重构的重点。我在这里想做的是：把读取行的 _做什么_ 与 _怎么做_ 分开，让代码对读者来说更具声明性。

虽然神奇数字 7 和 13 能完成工作，但它们的描述性不强。

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
)

func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[len(titleSeparator):]
	description := readLine()[len(descriptionSeparator):]

	return Post{Title: title, Description: description}, nil
}
```

现在我以创造性的重构思维盯着代码看，我想试试让 readLine 函数自己去掉 tag。还有一种更易读的方式可以把字符串前缀去掉，那就是 `strings.TrimPrefix`。

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
	}, nil
}
```

你可能喜欢也可能不喜欢这个想法，但我喜欢。重点是处于重构状态时，我们可以自由地玩弄内部细节，并可以一直跑测试来检查行为是否仍然正确。如果不满意，我们总可以回到之前的状态。TDD 方法给了我们这个频繁尝试想法的"许可证"，我们就有更多机会写出优秀的代码。

下一个需求是提取文章的 tags。如果你跟着我做的话，我建议你在继续阅读之前先自己尝试实现一下。你现在应该有了良好的、迭代式的节奏，对提取下一行并解析数据有信心。

为了简洁起见，我不再走 TDD 的步骤，下面是加上 tags 之后的测试。

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker`
	)

	// 为简洁起见省略其余测试代码
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
	})
}
```

如果你只是复制粘贴我写的内容，那是在欺骗自己。为了确保我们处于同一节奏，下面是我的代码，包括了提取 tags。

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
	tagsSeparator        = "Tags: "
)

func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
	}, nil
}
```

希望这里没有什么意外。我们能够复用 `readMetaLine` 来获取 tags 的下一行，然后用 `strings.Split` 把它们拆开。

正常路径的最后一次迭代是提取 body。

下面提醒一下我们提议的文件格式。

```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

The body of posts starts after the `---`
```

我们已经读了前 3 行。然后我们要再读一行，把它丢弃，文件剩下的部分就是文章的 body。

## 先写测试

修改测试数据加入分隔符，并加入一个含有几个换行的 body，以检查我们能抓到所有内容。

```go
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go
---
Hello
World`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker
---
B
L
M`
	)
```

像之前一样在断言里加上

```go
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
		Body: `Hello
World`,
	})
```

## 尝试运行测试

```
./blogpost_test.go:60:3: unknown field 'Body' in struct literal of type blogposts.Post
```

正如我们所料。

## 写最少的代码让测试能跑起来，并检查失败的测试输出

把 `Body` 加到 `Post` 上，测试应该会失败。

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:38: got {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:}, want {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:Hello
        World}
```

## 写足够的代码让它通过

1. 扫描下一行，忽略 `---` 分隔符。
2. 持续扫描直到没有数据可扫。

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	title := readMetaLine(titleSeparator)
	description := readMetaLine(descriptionSeparator)
	tags := strings.Split(readMetaLine(tagsSeparator), ", ")

	scanner.Scan() // 忽略一行

	buf := bytes.Buffer{}
	for scanner.Scan() {
		fmt.Fprintln(&buf, scanner.Text())
	}
	body := strings.TrimSuffix(buf.String(), "\n")

	return Post{
		Title:       title,
		Description: description,
		Tags:        tags,
		Body:        body,
	}, nil
}
```

* `scanner.Scan()` 返回一个 `bool`，表示是否还有更多数据可以扫描，所以我们可以用它配合 `for` 循环一直读到末尾。
* 每次 `Scan()` 之后，我们用 `fmt.Fprintln` 把数据写入缓冲区。我们使用会加换行的版本，因为 scanner 会把每一行的换行去掉，但我们需要保留它们。
* 由于上面的原因，我们需要把最后那个换行去掉，避免末尾有多余的换行。

## 重构

把"获取剩余数据"的概念封装到一个函数中，能帮助未来的读者快速理解 `newPost` 在 _做什么_，而不必关心实现细节。

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
		Body:        readBody(scanner),
	}, nil
}

func readBody(scanner *bufio.Scanner) string {
	scanner.Scan() // 忽略一行
	buf := bytes.Buffer{}
	for scanner.Scan() {
		fmt.Fprintln(&buf, scanner.Text())
	}
	return strings.TrimSuffix(buf.String(), "\n")
}
```

## 进一步迭代

我们已经做出了功能的"钢线"（steel thread），用最短路径达成了正常路径，但显然距离生产可用还有一段距离。

我们还没处理：

* 当文件格式不正确时
* 文件不是 `.md`
* 如果元数据字段的顺序不一样怎么办？应当允许吗？我们应该能处理吗？

但关键的是，我们已经有可以工作的软件了，并且定义了我们的接口。上面这些只是进一步的迭代，需要写更多的测试来驱动行为。要支持上述任何一项，我们都不需要改变 _设计_，只需改变实现细节。

专注于目标意味着我们做出了重要的决策，并依据期望行为验证了它们，而不是在不影响整体设计的事情上陷入泥潭。

## 总结

`fs.FS` 以及 Go 1.16 中的其他变化，给了我们从文件系统读取数据并简单测试的优雅方式。

如果你想"真正"试试这段代码：

* 在项目里创建一个 `cmd` 文件夹，新建一个 `main.go` 文件
* 加入下面的代码

```go
import (
	blogposts "github.com/quii/fstest-spike"
	"log"
	"os"
)

func main() {
	posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
	if err != nil {
		log.Fatal(err)
	}
	log.Println(posts)
}
```

* 在 `posts` 文件夹里加入一些 markdown 文件，运行程序！

注意生产代码

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

和测试

```go
posts, err := blogposts.NewPostsFromFS(fs)
```

之间的对称性。

这就是以使用者为驱动、自顶向下的 TDD _感觉对了_ 的时候。

我们包的使用者可以查看我们的测试，迅速搞明白它应该做什么以及怎么用。作为维护者，我们可以 _对我们的测试有信心，因为它们是从使用者的视角写出来的_。我们不是在测试实现细节或其他无关细节，所以可以合理地相信我们的测试在重构时会帮我们而不是阻碍我们。

通过依赖良好的软件工程实践，比如[**依赖注入**](dependency-injection.md)，我们的代码很容易测试和复用。

当你创建包的时候，即使它们只是项目内部使用，也优先采用自顶向下、以使用者为驱动的方法。这能阻止你过度想象设计、做出可能根本用不上的抽象，并能确保你写的测试是有用的。

迭代式的方法让每一步都很小，持续的反馈帮助我们比那些更随意的方式更早地揭示出不清晰的需求。

### 写入呢？

需要注意的是，这些新特性只有 _读取_ 文件的操作。如果你的工作需要写入，你得另寻他法。记得继续思考标准库目前提供了什么——如果你在写入数据，你应该研究利用现有的接口，例如 `io.Writer`，让你的代码保持松耦合和可复用。

### 进一步阅读

* 这只是对 `io/fs` 的一个简单介绍。[Ben Congdon 写了一篇出色的文章](https://benjamincongdon.me/blog/2021/01/21/A-Tour-of-Go-116s-iofs-package/)，对本章的写作帮助很大。
* [关于文件系统接口的讨论](https://github.com/golang/go/issues/41190)
