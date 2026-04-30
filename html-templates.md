# HTML 模板

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/blogrenderer)**

我们生活在这样一个世界：每个人都想用当下最流行的前端框架来构建 web 应用，背后是几十亿字节的转译后 JavaScript，配合一个拜占庭式复杂的构建系统；[但也许这并不总是必要](https://quii.dev/The_Web_I_Want)。

我会说大多数 Go 开发者都重视简单、稳定且快速的工具链，而前端世界在这方面经常令人失望。

很多网站不需要做成 [SPA](https://en.wikipedia.org/wiki/Single-page_application)。**HTML 和 CSS 是非常棒的内容传输方式**，你可以用 Go 做一个网站来交付 HTML。

如果你仍然希望有一些动态元素，可以加入一些客户端 JavaScript，或者你也可以试试 [Hotwire](https://hotwired.dev)，它能让你以服务端为主的方式提供动态体验。

你可以通过精巧地使用 [`fmt.Fprintf`](https://pkg.go.dev/fmt#Fprintf) 在 Go 中生成 HTML，但本章你将学到 Go 标准库中以更简单、更易维护的方式生成 HTML 的工具。你还会学到一些你之前可能没用过的、对这类代码更有效的测试方式。

## 我们要构建什么

在 [读取文件](/reading-files.md) 那一章，我们写了一些代码：接受一个 [`fs.FS`](https://pkg.go.dev/io/fs)（一个文件系统），并为遇到的每个 markdown 文件返回一个 `Post` 切片。

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

下面是我们对 `Post` 的定义

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

下面是一个可以被解析的 markdown 文件示例。

```markdown
Title: Welcome to my blog
Description: Introduction to my blog
Tags: cooking, family, live-laugh-love
---
# First recipe!
Welcome to my **amazing recipe blog**. I am going to write about my family recipes, and make sure I write a long, irrelevant and boring story about my family before you get to the actual instructions.
```

如果我们继续编写博客软件的旅程，我们会拿着这些数据，从中生成 HTML，让我们的 web 服务器作为 HTTP 请求的响应返回。

对于我们的博客，我们想生成两种页面：

1. **查看文章**。渲染特定的一篇文章。`Post` 中的 `Body` 字段是包含 markdown 的字符串，所以应当被转换成 HTML。
2. **首页**。列出所有文章，每篇都有指向文章详情页的超链接。

我们也会希望整个站点拥有一致的外观和体验，所以每个页面都会有常见的 HTML 结构，比如 `<html>` 和 `<head>`，`<head>` 中包含 CSS 样式表的链接以及我们想要的其他东西。

构建博客软件时，在如何构建并把 HTML 发送到用户浏览器方面，你有几种方式可以选。

我们设计代码时让它接收一个 `io.Writer`。这意味着我们代码的调用方有以下灵活性：

- 把它们写到 [os.File](https://pkg.go.dev/os#File)，从而可以静态地提供
- 直接把 HTML 写到 [`http.ResponseWriter`](https://pkg.go.dev/net/http#ResponseWriter)
- 或者真的写到任何东西！只要它实现了 `io.Writer`，用户就能从一个 `Post` 生成一些 HTML

## 先写测试

一如既往，在过早地动手前思考一下需求很重要。我们如何把这个相对庞大的需求集合，拆解成一个小巧、可达成的步骤来聚焦？

在我看来，实际查看内容比首页更高优。我们可以先发布这个产品，然后分享指向我们精彩内容的直接链接。一个不能链接到实际内容的首页没什么用。

不过，按前文描述渲染一篇文章感觉还是太大。所有 HTML 结构，把 body 里的 markdown 转成 HTML，列出标签，等等。

在这个阶段我并不太关心具体的标签结构，最容易上手的第一步就是检查我们能不能把文章的标题渲染成 `<h1>`。这 _感觉_ 像是能让我们前进一小步的最小步骤。

```go
package blogrenderer_test

import (
	"bytes"
	"github.com/quii/learn-go-with-tests/blogrenderer"
	"testing"
)

func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>`
		if got != want {
			t.Errorf("got '%s' want '%s'", got, want)
		}
	})
}
```

我们决定接收 `io.Writer` 也让测试变得简单，本例中我们写到一个 [`bytes.Buffer`](https://pkg.go.dev/bytes#Buffer)，之后可以检查它的内容。

## 尝试运行测试

如果你已经读过本书前面的章节，对此应该相当熟练了。你不能运行测试，因为我们还没定义这个包，也没有 `Render` 函数。试着自己跟着编译器的提示走，让代码处于一个能跑测试、并看到带清晰错误信息的失败状态。

让你的测试真正经历失败这一步非常重要，将来某天你不小心让一个测试失败时，你会感谢自己 _现在_ 花了功夫确认它失败时有清晰的错误信息。

## 写最少量的代码让测试能跑起来，并查看失败的测试输出

下面是让测试能跑起来的最少代码

```go
package blogrenderer

// 如果你是从读取文件那一章接着过来的，不应该重复定义这个
type Post struct {
	Title, Description, Body string
	Tags                     []string
}

func Render(w io.Writer, p Post) error {
	return nil
}
```

测试应该会抱怨空字符串不等于我们想要的内容。

## 写足够的代码让测试通过

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1>", p.Title)
	return err
}
```

记住，软件开发主要是一种学习活动。为了在工作中不断发现与学习，我们需要以一种能产生频繁、高质量反馈循环的方式工作，而最简单的方式就是用小步骤工作。

所以我们现在不去考虑使用任何模板库。仅靠"普通的"字符串拼接其实就能很好地构造 HTML，跳过模板这部分的同时我们能验证一小块有用的行为，并对我们包的 API 做了一点点设计工作。

## 重构

目前没什么可重构的，那么进入下一次迭代

## 先写测试

我们已经有了非常基础的工作版本，可以在测试上迭代来扩展功能。在这个例子中，从 `Post` 渲染更多信息。

```go
	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>
<p>This is a description</p>
Tags: <ul><li>go</li><li>tdd</li></ul>`

		if got != want {
			t.Errorf("got '%s' want '%s'", got, want)
		}
	})
```

注意这样写 _感觉_ 很别扭。看到测试里塞了那么多标签让人难受，而且我们甚至还没把 body 加进去，也还没加上我们想要的整个 HTML，比如所有 `<head>` 内容以及所需的页面结构。

不过我们 _暂时_ 先忍着痛苦。

## 尝试运行测试

它应该会失败，抱怨它没有我们期望的字符串，因为我们没有渲染 description 和 tags。

## 写足够的代码让测试通过

试着自己做一下，而不是拷贝代码。你会发现让这个测试通过 _有点烦人_！我尝试时第一次得到的错误是

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:32: got '<h1>hello world</h1><p>This is a description</p><ul><li>go</li><li>tdd</li></ul>' want '<h1>hello world</h1>
        <p>This is a description</p>
        Tags: <ul><li>go</li><li></li></ul>'
```

换行！谁在乎呢？嗯，我们的测试在乎，因为它在做精确字符串匹配。它应该这样吗？我现在先把换行去掉只为让测试通过。

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1><p>%s</p>", p.Title, p.Description)
	if err != nil {
		return err
	}

	_, err = fmt.Fprint(w, "Tags: <ul>")
	if err != nil {
		return err
	}

	for _, tag := range p.Tags {
		_, err = fmt.Fprintf(w, "<li>%s</li>", tag)
		if err != nil {
			return err
		}
	}

	_, err = fmt.Fprint(w, "</ul>")
	if err != nil {
		return err
	}

	return nil
}
```

**哎哟**。这不是我写过最漂亮的代码，而且我们的标签实现还非常初级。我们的页面会需要远比这更多的内容和元素，我们很快就看出这种方式不合适。

但关键是，我们有了一个通过的测试；我们有了能工作的软件。

## 重构

有了通过测试这个安全网，我们可以在重构阶段考虑改变实现方式了。

### 引入模板

Go 有两个模板包 [text/template](https://pkg.go.dev/text/template) 和 [html/template](https://pkg.go.dev/html/template)，它们共享同一个接口。它们都做的事是允许你把模板和数据组合起来生成字符串。

HTML 版本有什么不同？

> 包 template (html/template) 实现了数据驱动的模板，用于生成可安全防御代码注入的 HTML 输出。它提供了与 text/template 相同的接口，无论何时输出是 HTML，都应使用它来代替 text/template。

模板语言和 [Mustache](https://mustache.github.io) 非常相似，能让你以一种很整洁的方式动态生成内容，并很好地分离关注点。相比你可能用过的其他模板语言，它非常受限，或者按 Mustache 的说法是"逻辑无关"的。这是一项重要的、**有意为之**的设计决策。

虽然我们这里聚焦的是生成 HTML，但如果你的项目在做复杂的字符串拼接和折腾，你可能会想用 `text/template` 来整理你的代码。

### 回到代码

下面是我们博客的一个模板：

`<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`

我们在哪里定义这个字符串呢？嗯，有几个选择，但为了保持步骤小，我们就先从一个普普通通的字符串开始

```go
package blogrenderer

import (
	"html/template"
	"io"
)

const (
	postTemplate = `<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`
)

func Render(w io.Writer, p Post) error {
	templ, err := template.New("blog").Parse(postTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

我们用一个名字创建一个新模板，然后解析模板字符串。然后我们就能在它上面调用 `Execute` 方法，传入数据，本例中是 `Post`。

模板会把 `{{.Description}}` 之类的东西替换成 `p.Description` 的内容。模板还提供了一些编程原语，比如 `range` 用来循环遍历值，以及 `if`。你可以在 [text/template 文档](https://pkg.go.dev/text/template) 中找到更多细节。

_这应该是一次纯重构。_ 我们不需要改测试，它们应该继续通过。重要的是，我们的代码更易读，也少了很多烦人的错误处理。

人们经常抱怨 Go 中错误处理的啰嗦，但你或许会发现你能找到更好的方式来写代码，让它从一开始就不那么容易出错，就像这里一样。

### 进一步重构

使用 `html/template` 绝对是一种改进，但把它作为字符串常量放在我们的代码里并不理想：

- 它仍然挺难读的。
- 对 IDE/编辑器不友好。没有语法高亮、没法重新格式化、重构等等。
- 它看起来像 HTML，但你没法像处理"普通的" HTML 文件那样去处理它

我们想做的是把模板放到独立的文件里，这样我们可以更好地组织它们，并像处理 HTML 文件一样处理它们。

创建一个名为 "templates" 的文件夹，在里面新建一个文件 `blog.gohtml`，把我们的模板粘贴到这个文件里。

现在修改我们的代码，使用 [Go 1.16 引入的 embed 功能](https://pkg.go.dev/embed) 来嵌入文件系统。

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

通过把"文件系统"嵌入到我们的代码中，我们可以加载多个模板并自由组合。当我们想在不同模板之间共享渲染逻辑时（比如 HTML 页面顶部的 header 和 footer），这会变得很有用。

### Embed？

Embed 在 [读取文件](reading-files.md) 中有简短地接触过。[标准库的文档解释](https://pkg.go.dev/embed)

> 包 embed 提供对嵌入到运行中的 Go 程序中的文件的访问。
>
> 导入了 "embed" 的 Go 源文件可以使用 //go:embed 指令，用编译期从包目录或子目录读取的文件内容，初始化类型为 string、[]byte 或 FS 的变量。

我们为什么要用这个？另一种方式是我们 _可以_ 从"普通的"文件系统加载模板。但这意味着无论我们想在哪里使用这个软件，都得保证模板放在正确的文件路径上。在你的工作中，你可能会有不同的环境，比如开发、预发和生产。要让它工作，你得确保模板被复制到正确的位置。

使用 embed 的话，文件会在你构建程序时被包含进去。这意味着一旦你构建完程序（你只该构建一次），这些文件总是可用的。

更方便的是，你不仅可以嵌入单个文件，还可以嵌入文件系统；并且这个文件系统实现了 [io/fs](https://pkg.go.dev/io/fs)，这意味着你的代码不需要关心它在和哪种文件系统打交道。

不过如果你希望根据配置使用不同的模板，那你可能会希望坚持用更常规的方式从磁盘加载模板。

## 接下来：让模板"漂亮"起来

我们并不希望我们的模板被定义成一行字符串。我们希望把它分行排开，让它更易读、更好维护，类似下面这样：

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

但如果我们这么做，测试就会失败。这是因为我们的测试在期望返回一个非常具体的字符串。

但说真的，我们其实并不在乎空白字符。如果每次对标签做点小改动都得费力地更新断言字符串，维护这个测试会变成一场噩梦。随着模板增长，这种修改会变得更难管理，工作成本会失控。

## 引入审批测试（Approval Tests）

[Go Approval Tests](https://github.com/approvals/go-approval-tests)

> ApprovalTests 让你能轻松测试较大的对象、字符串以及任何能保存到文件的东西（图像、声音、CSV 等等……）

这个想法和"golden 文件"或快照测试类似。它不是让你在测试文件里费劲地维护字符串，而是让审批工具帮你将输出和你创建的"已批准"文件做比较。然后你只要在批准它后简单地把新版本拷贝过去即可。重新跑测试，你又回到绿色了。

把 `"github.com/approvals/go-approval-tests"` 添加为项目依赖，并把测试改成下面这样

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := blogrenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

第一次运行它时会失败，因为我们还没批准任何东西

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:29: Failed Approval: received does not match approved.
```

它会创建两个文件，类似下面这样

- `renderer_test.TestRender.it_converts_a_single_post_into_HTML.received.txt`
- `renderer_test.TestRender.it_converts_a_single_post_into_HTML.approved.txt`

received 文件里是新的、未批准的输出。把它的内容复制到空的 approved 文件里，然后重新运行测试。

通过复制新版本，你"批准"了这次更改，测试现在通过了。

为了直观看到这个工作流，把模板改成我们之前讨论的、更易读的样子（在语义上是一样的）。

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

重新运行测试。会生成一个新的 "received" 文件，因为我们代码的输出和已批准的版本不同了。看一下，如果你对修改满意，简单地把新版本覆盖过去并重新运行测试。一定要把已批准的文件提交到版本控制中。

这种方式让管理 HTML 这种又大又丑的内容的变更简单得多。你可以用 diff 工具来查看和管理差异，并且让你的测试代码保持干净。

![用 diff 工具管理变更](https://i.imgur.com/0MoNdva.png)

这其实只是审批测试的一种很轻量的用法，它在你的测试军火库里是个非常有用的工具。[Emily Bache](https://twitter.com/emilybache) 有一段 [有趣的视频，她在视频里用审批测试为一个零测试的复杂代码库添加了一套极其全面的测试](https://www.youtube.com/watch?v=zyM2Ep28ED8)。"组合测试"（Combinatorial Testing）绝对值得了解。

做完这个改动后，我们仍然能从代码良好测试中受益，但当我们在折腾标签时，测试也不会过多妨碍我们了。

### 我们还在做 TDD 吗？

这种方法的一个有趣副作用是，它把我们带离了 TDD。当然你 _可以_ 手动把已批准文件改成你想要的状态、运行测试，然后修改模板让它输出你定义的内容。

但这就太蠢了！TDD 是一种工作方式，特别是用于设计；但这并不意味着我们必须教条地把它用于 **所有** 事情。

重要的是，我们做了正确的事，把 TDD 当作一种 **设计工具** 来设计我们包的 API。对于模板的修改我们的流程可以是：

- 对模板做一个小改动
- 跑审批测试
- 用眼睛看一眼输出，检查是否正确
- 做出审批
- 重复

我们仍然不应该放弃以小而可达成的步骤工作的价值。试着想办法让改动小一点，不断重新跑测试以获得对当前所做事情的真实反馈。

如果我们开始改的是模板 _周围_ 的代码，那当然可能值得回到 TDD 的工作方式。

## 扩展标签

大多数网站的 HTML 比我们现在的要丰富得多。首先有 `html` 元素，再加上 `head`，可能还有 `nav`。通常还会有 footer 的概念。

如果我们的站点要有不同的页面，我们会希望把这些东西定义在一处，让站点保持一致的外观。Go 模板支持我们定义片段，然后在其他模板中导入。

修改我们已有的模板，导入一个顶部和底部模板

```handlebars
{{template "top" .}}
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
{{template "bottom" .}}
```

然后用下面的内容创建 `top.gohtml`

```handlebars
{{define "top"}}
<!DOCTYPE html>
<html lang="en">
<head>
    <title>My amazing blog!</title>
    <meta charset="UTF-8"/>
    <meta name="description" content="Wow, like and subscribe, it really helps the channel guys" lang="en"/>
</head>
<body>
<nav role="navigation">
    <div>
        <h1>Budding Gopher's blog</h1>
        <ul>
            <li><a href="/">home</a></li>
            <li><a href="about">about</a></li>
            <li><a href="archive">archive</a></li>
        </ul>
    </div>
</nav>
<main>
{{end}}
```

以及 `bottom.gohtml`

```handlebars
{{define "bottom"}}
</main>
<footer>
    <ul>
        <li><a href="https://twitter.com/quii">Twitter</a></li>
        <li><a href="https://github.com/quii">GitHub</a></li>
    </ul>
</footer>
</body>
</html>
{{end}}
```

（显然，你想放什么标签都行！）

我们现在需要指定一个具体的模板来运行。在 blog renderer 中，把 `Execute` 命令改成 `ExecuteTemplate`

```go
if err := templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
	return err
}
```

重新运行你的测试。会生成一个新的 "received" 文件，测试会失败。看一下，如果你满意，把它覆盖到旧版本上来批准它。再次运行测试，应该通过了。

## 顺便玩一下基准测试

继续之前，我们来思考一下我们的代码做了什么。

```go
func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	if err := templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
		return err
	}

	return nil
}
```

- 解析模板
- 用模板把一篇文章渲染到 `io.Writer`

虽然在大多数情况下，每篇文章都重新解析模板对性能的影响相当微小，但 _不_ 这么做的代价也很小，而且应该能让代码稍微整洁一些。

为了直观地看到不重复解析的影响，我们可以用基准测试工具看看我们的函数有多快。

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	for b.Loop() {
		blogrenderer.Render(io.Discard, aPost)
	}
}
```

在我的电脑上，结果如下

```
BenchmarkRender-8 22124 53812 ns/op
```

为了避免一遍又一遍重新解析模板，我们创建一个类型来持有解析好的模板，并在它上面定义一个方法来做渲染

```go
type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {

	if err := r.templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
		return err
	}

	return nil
}
```

这改变了我们代码的接口，所以需要更新测试

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		t.Fatal(err)
	}

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := postRenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

以及我们的基准测试

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		b.Fatal(err)
	}

	for b.Loop() {
		postRenderer.Render(io.Discard, aPost)
	}
}
```

测试应该继续通过。那基准测试呢？

`BenchmarkRender-8 362124 3131 ns/op`。之前的 ns/op 是 `53812 ns/op`，所以这是个相当不错的提升！当我们再添加其他渲染方法（比如首页）时，因为不需要重复解析模板，代码也会更简洁。

## 回到正事

在渲染文章这件事上，剩下重要的部分其实是渲染 `Body`。如果你还记得，那应该是作者写的 markdown，所以需要转换成 HTML。

我们把这个留作给读者你的练习。你应该能找到一个 Go 库来帮你做这件事。用审批测试来验证你做的事情。

### 关于测试第三方库

**注意**。要小心，不要在单元测试里过度关注于显式测试某个第三方库的行为。

针对你不掌控的代码写测试是一种浪费，并增加维护负担。有时你可能会希望使用 [依赖注入](./dependency-injection.md) 来控制一个依赖，并在测试中 mock 它的行为。

不过在本例中，我把 markdown 转 HTML 视为渲染的实现细节，我们的审批测试应该能给我们足够的信心。

### 渲染首页

我们接下来要做的功能是渲染一个首页，把文章列成一个 HTML 有序列表。

我们在扩展 API，所以重新戴上 TDD 的帽子。

## 先写测试

表面上，首页似乎很简单，但写测试仍然会促使我们做出一些设计选择

```go
t.Run("it renders an index of posts", func(t *testing.T) {
	buf := bytes.Buffer{}
	posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

	if err := postRenderer.RenderIndex(&buf, posts); err != nil {
		t.Fatal(err)
	}

	got := buf.String()
	want := `<ol><li><a href="/post/hello-world">Hello World</a></li><li><a href="/post/hello-world-2">Hello World 2</a></li></ol>`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
})
```

1. 我们把 `Post` 的 title 字段作为 URL 路径的一部分，但我们不想 URL 里有空格，所以用连字符替换它们。
2. 我们给 `PostRenderer` 加了一个 `RenderIndex` 方法，同样接收一个 `io.Writer` 和一个 `Post` 切片。

如果我们坚持先写代码后写测试，配合审批测试方式，我们就不会在一个受控环境中回答这些问题。**测试给了我们思考的空间**。

## 尝试运行测试

```
./renderer_test.go:41:13: undefined: blogrenderer.RenderIndex
```

## 写最少量的代码让测试能跑起来，并查看失败的测试输出

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return nil
}
```

上面的代码应该让测试出现下面的失败

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
--- FAIL: TestRender (0.00s)
```

## 写足够的代码让测试通过

虽然这件事 _感觉_ 应该容易，但其实有点别扭。我分了好几步来做

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

我一开始不想折腾独立的模板文件，只想先让它工作起来。我把"先解析"和"分文件"视为以后可以做的重构。

这没通过，但已经很接近了。

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "<ol><li><a href=\"/post/Hello%20World\">Hello World</a></li><li><a href=\"/post/Hello%20World%202\">Hello World 2</a></li></ol>" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
--- FAIL: TestRender (0.00s)
    --- FAIL: TestRender/it_renders_an_index_of_posts (0.00s)
```

你可以看到模板代码把 `href` 属性中的空格做了转义。我们需要一种方式把空格替换成连字符。我们不能直接遍历 `[]Post` 在内存里替换它们，因为我们仍然希望显示给用户的链接锚文本里有空格。

我们有几个选择。第一个我们要探索的是把一个函数传给模板。

### 把函数传入模板

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{sanitiseTitle .Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Funcs(template.FuncMap{
		"sanitiseTitle": func(title string) string {
			return strings.ToLower(strings.Replace(title, " ", "-", -1))
		},
	}).Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

_在你解析模板之前_，你可以给模板加一个 `template.FuncMap`，它允许你定义可以在模板中调用的函数。本例中我们做了一个 `sanitiseTitle` 函数，然后我们在模板里用 `{{sanitiseTitle .Title}}` 来调用它。

这是一项强大的特性，把函数送入模板能让你做一些非常酷的事，但是，你应该这样吗？回到 Mustache 和逻辑无关模板的原则，他们为什么主张逻辑无关？**模板里的逻辑有什么问题？**

正如我们已经看到的，为了测试我们的模板，_我们不得不引入了一种完全不同的测试方式_。

想象一下你往模板里塞了一个有几种不同行为分支和边界情况的函数，**你怎么测它**？以目前的设计，你测这种逻辑的唯一方式就是 _渲染 HTML 并比较字符串_。这并不是一种容易或理智的逻辑测试方式，绝对不是你想给 _重要的_ 业务逻辑用的方式。

虽然审批测试技术降低了维护这些测试的成本，但它们仍然比你写的大多数单元测试更难维护。它们对你做出的任何小标签改动仍然敏感，只是我们让管理变得简单了一些。我们仍然应该努力把代码组织好，让我们不必围绕模板写很多测试，并尽量把不需要在渲染代码里的逻辑剥离出去。

受 Mustache 影响的模板引擎给你的是一种有用的约束，不要太频繁地试图绕开它；**不要逆水行舟**。相反，拥抱 [视图模型（view models）](https://stackoverflow.com/a/11074506/3193) 的思路：构造特定类型，让其包含渲染所需的数据，且其形态对模板语言来说很方便。

这样一来，无论你用怎样重要的业务逻辑生成那一包数据，都可以独立地做单元测试，远离 HTML 和模板这片混乱地带。

### 分离关注点

那我们可以做些什么呢？

#### 给 `Post` 加一个方法，然后在模板中调用

我们可以在模板代码里调用我们传入类型的方法，因此可以给 `Post` 加一个 `SanitisedTitle` 方法。这样能简化模板，并且我们想的话也很容易单独单元测试这部分逻辑。这大概是最简单的方案，虽然不一定是最简洁的。

这种方式的一个缺点是，这仍然是 _视图_ 逻辑。它对系统的其他部分没什么用，但现在却成了核心领域对象 API 的一部分。这种做法日积月累，可能会让你创造出 [上帝对象](https://en.wikipedia.org/wiki/God_object)。

#### 创建一个专门的视图模型类型，比如 `PostViewModel`，里面只放我们需要的数据

我们的渲染代码不再耦合于领域对象 `Post`，而是接收一个视图模型。

```go
type PostViewModel struct {
	Title, SanitisedTitle, Description, Body string
	Tags                                     []string
}
```

我们代码的调用方需要把 `[]Post` 映射为 `[]PostView`，并生成 `SanitizedTitle`。一个保持整洁的方式是有一个 `func NewPostView(p Post) PostView` 来封装这种映射。

这能让我们的渲染代码保持逻辑无关，也是我们能做到的最严格的关注点分离，但代价是渲染文章的过程稍微更曲折一些。

两种方式都可以，本例中我倾向于选第一种。在系统演化时，你应该警惕仅仅为了让渲染顺畅就不断添加各种零散的方法；当领域对象到视图的转换变得更复杂时，专门的视图模型会更有用。

那我们就可以给 `Post` 加上方法

```go
func (p Post) SanitisedTitle() string {
	return strings.ToLower(strings.Replace(p.Title, " ", "-", -1))
}
```

然后我们的渲染代码就能回归一个更简单的世界

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

## 重构

终于测试应该通过了。我们现在可以把模板挪到一个文件里（`templates/index.gohtml`），并在构造 renderer 时一次性加载。

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", p)
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}
```

由于把多个模板都解析到了 `templ` 里，我们现在必须调用 `ExecuteTemplate` 并指定 _要_ 渲染哪个模板，但希望你也同意，我们最终得到的代码看起来很棒。

如果有人重命名其中一个模板文件，会引入一个 _轻微_ 的 bug 风险，但我们快速运行的单元测试会很快捕捉到它。

现在我们对包的 API 设计满意了，并通过 TDD 推导出了一些基本行为，让我们把测试改成使用审批测试。

```go
	t.Run("it renders an index of posts", func(t *testing.T) {
		buf := bytes.Buffer{}
		posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

		if err := postRenderer.RenderIndex(&buf, posts); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
```

记得运行测试看到它失败，然后批准这次更改。

最后我们可以给首页加上页面结构：

```handlebars
{{template "top" .}}
<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>
{{template "bottom" .}}
```

重新运行测试，批准更改，首页就完成了！

## 渲染 markdown body

我之前鼓励你自己尝试，下面是我最终采用的方式。

```go
package blogrenderer

import (
	"embed"
	"github.com/gomarkdown/markdown"
	"github.com/gomarkdown/markdown/parser"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ    *template.Template
	mdParser *parser.Parser
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	extensions := parser.CommonExtensions | parser.AutoHeadingIDs
	parser := parser.NewWithExtensions(extensions)

	return &PostRenderer{templ: templ, mdParser: parser}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", newPostVM(p, r))
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}

type postViewModel struct {
	Post
	HTMLBody template.HTML
}

func newPostVM(p Post, r *PostRenderer) postViewModel {
	vm := postViewModel{Post: p}
	vm.HTMLBody = template.HTML(markdown.ToHTML([]byte(p.Body), r.mdParser, nil))
	return vm
}
```

我用了出色的 [gomarkdown](https://github.com/gomarkdown/markdown) 库，它的工作方式正如我所希望的。

如果你自己尝试过，可能会发现你 body 的渲染结果中 HTML 被转义了。这是 Go 的 html/template 包的一个安全特性，用来阻止恶意第三方 HTML 被输出。

要绕开这一点，在你发送到 render 的类型里，需要把你信任的 HTML 包装在 [template.HTML](https://pkg.go.dev/html/template#HTML) 中

> HTML 封装了一段已知安全的 HTML 文档片段。它不应被用于来自第三方的 HTML，或带有未关闭标签或注释的 HTML。来自健全的 HTML 净化器以及由本包做了转义的模板的输出，可以放心地以这种类型使用。
>
> 使用此类型存在安全风险：被封装的内容应来自可信源，因为它会被原封不动地包含在模板输出中。

所以我创建了一个 **未导出** 的视图模型（`postViewModel`），因为我仍把它视为渲染的内部实现细节。我没必要单独测试它，也不希望它污染我的 API。

我在渲染时构造一个，用来把 `Body` 解析为 `HTMLBody`，然后我在模板里使用这个字段来渲染 HTML。

## 总结

如果你结合 [读取文件](reading-files.md) 那章和这一章的所学，你可以舒服地做出一个有良好测试、简单的静态站点生成器，并搭起你自己的博客。再找一些 CSS 教程，你也能让它看起来不错。

这种方法不止适用于博客。从任意来源——数据库、API 还是文件系统——拿数据，转成 HTML 并从服务器返回，是一种跨越数十年的简单技术。人们喜欢抱怨现代 web 开发的复杂，但你确定你不是只是在自找麻烦吗？

Go 非常适合 web 开发，特别是当你能清醒思考你正在做的网站的真实需求时。在服务端生成 HTML，往往是比用 React 这类技术做"web 应用"更好、更简单、性能也更好的方式。

### 我们学到了什么

- 如何创建并渲染 HTML 模板。
- 如何把模板组合在一起，[DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself) 化相关的标签，帮助我们保持一致的外观体验。
- 如何把函数传入模板，以及为什么你应该再三考虑这样做。
- 如何写"审批测试"，它能帮我们测试模板渲染器这种又大又丑的输出。

### 关于逻辑无关的模板

像往常一样，这一切都是关于 **关注点分离**。重要的是我们要思考系统各部分的职责是什么。人们经常把重要的业务逻辑漏到模板里，混淆了关注点，让系统难以理解、维护和测试。

### 不仅是 HTML

记住 Go 还有 `text/template` 用来从模板生成其他类型的数据。如果你发现自己需要把数据转换成某种结构化输出，本章介绍的技巧也可以派上用场。

### 参考资料和延伸阅读

- [John Calhoun 的 'Learn Web Development with Go'](https://www.calhoun.io/intro-to-templates-p1-contextual-encoding/) 有许多关于模板的优秀文章。
- [Hotwire](https://hotwired.dev) - 你可以用本章的技巧创建 Hotwire web 应用。它由 Basecamp 开发，他们主要是 Ruby on Rails 团队，但因为它是服务端的，我们可以在 Go 中使用它。
