# Learn Go with Tests - 扩展验收测试（以及 gRPC 简单入门）

本章是 [验收测试入门](https://quii.gitbook.io/learn-go-with-tests/testing-fundamentals/intro-to-acceptance-tests) 的延续。你可以在 [GitHub 上找到本章的完整代码](https://github.com/quii/go-specs-greet)。

验收测试至关重要，它直接影响你能否随着时间推移自信地演进系统，并保持合理的变更成本。

它们也是处理遗留代码的绝佳工具。当你面对一个糟糕的、没有任何测试的代码库时，请抵制立刻开始重构的冲动。相反，先写一些验收测试，给自己一张安全网，然后就可以自由地修改系统内部，而不影响它对外的功能行为。验收测试不必关心内部质量，因此在这种情况下非常合适。

读完本章后你会体会到，验收测试不仅可用于验证，还能在开发过程中帮助我们更深思熟虑、更有条理地改变系统，减少无谓的浪费。

## 前置材料

本章的灵感来自我多年来对验收测试的挫折感。我推荐你看下面两个视频：

- Dave Farley - [How to write acceptance tests](https://www.youtube.com/watch?v=JDD5EEJgpHU)
- Nat Pryce - [E2E functional tests that can run in milliseconds](https://www.youtube.com/watch?v=Fk4rCn4YLLU)

《Growing Object Oriented Software》（GOOS）对包括我在内的许多软件工程师来说都是一本非常重要的书。它推崇的方法正是我教导同事们去遵循的方法。

- [GOOS](http://www.growing-object-oriented-software.com) - Nat Pryce & Steve Freeman

最后，[Riya Dattani](https://twitter.com/dattaniriya) 和我在演讲 [Acceptance tests, BDD and Go](https://www.youtube.com/watch?v=ZMWJCk_0WrY) 中，结合 BDD 探讨过这个话题。

## 回顾

我们讨论的是"黑盒"测试，它从外部、从"**业务视角**"验证你的系统按预期行为运作。这些测试无法访问被测系统的内部细节；它们只关心你的系统**做什么**，而不关心**怎么做**。

## 糟糕验收测试的"解剖"

多年来，我在多家公司、多个团队工作过。每一家都意识到需要验收测试，需要某种方式从用户视角去测试系统并验证它按预期工作，但几乎无一例外，这些测试的成本对团队来说都成为了真正的问题。

- 运行慢
- 脆弱
- 不稳定
- 维护成本高，似乎让修改软件变得比应有的更困难
- 只能在特定环境运行，导致缓慢且糟糕的反馈循环

假设你打算为正在构建的网站写验收测试。你决定使用一个无头浏览器（比如 [Selenium](https://www.selenium.dev)）模拟用户点击网站上的按钮，验证它达到预期。

随着时间推移，你的网站标记会因为新功能而被修改，工程师们会为某个东西到底应该是 `<article>` 还是 `<section>` 第无数次地争吵不休。

即使团队对系统的改动很小、用户几乎察觉不到，你却发现自己花费大量时间更新验收测试。

### 紧耦合

想想是什么会触发验收测试需要修改：

- 外部行为变更。如果你想改变系统做的事，相应地改变验收测试套件似乎是合理的，甚至是期望的。
- 实现细节变更 / 重构。理想情况下这不应该触发修改，即使要改也只是小幅度的。

然而很多时候，正是后者让验收测试不得不改。以至于工程师因为顾虑更新测试的工作量而不愿意修改系统！

![Riya 和我谈论在测试中分离关注点](https://i.imgur.com/bbG6z57.png)

这些问题源于没有运用上面提到的作者们所写下、被广泛实践且被验证的工程习惯。**你不能像写单元测试那样写验收测试**；它们需要更多思考和不同的实践方式。

## 好验收测试的"解剖"

如果我们想要的验收测试只在行为变化时才需要修改，而不会因为实现细节变化而修改，那么我们就需要把这两类关注点分离开来。

### 关于复杂度的种类

作为软件工程师，我们要面对两种复杂度。

- **意外复杂度（Accidental complexity）**是因为我们要和计算机打交道而带来的复杂度，比如网络、磁盘、API 等等。

- **本质复杂度（Essential complexity）**有时也叫"领域逻辑"。它是你所在领域中的特定规则和真理。
  - 例如，"如果账户持有人取出的金额超过可用余额，他们就处于透支状态"。这句话和计算机毫无关系；甚至在银行使用计算机之前，它就是真的！

本质复杂度应该能用非技术人员的语言表达出来，把它在我们的"领域"代码以及验收测试中建模是很有价值的。

### 关注点分离

Dave Farley 在前面提到的视频中提出的，以及 Riya 和我也讨论过的，是我们应该有**规约（specifications）**的概念。规约描述了我们想要的系统行为，而不与意外复杂度或实现细节耦合。

这个想法对你来说应该是合理的。在生产代码里，我们经常努力分离关注点、解耦工作单元。你会犹豫要不要引入一个 `interface` 把你的 `HTTP` handler 和非 HTTP 关注点解耦吗？让我们用同样的思路对待验收测试。

Dave Farley 描述了一种具体的结构。

![Dave Farley 谈论验收测试](https://i.imgur.com/nPwpihG.png)

在 GopherconUK 上，Riya 和我用 Go 的术语重新表达了它。

![关注点分离](https://i.imgur.com/qdY4RJe.png)

### 强化版的测试

把规约的执行方式解耦出来，可以让我们在不同场景下复用它。我们可以：

#### 让我们的驱动可配置

这意味着你可以在本地、staging 和（理想情况下）生产环境运行验收测试。
- 太多团队设计的系统让验收测试无法在本地运行。这会引入慢得让人无法忍受的反馈循环。难道你不更希望在 _合并代码之前_ 就有信心验收测试会通过吗？如果测试开始挂了，难道你能接受没法在本地复现失败，而只能提交代码、然后双手合十祈祷它能在 20 分钟后的另一个环境中通过吗？
- 记住，仅仅因为你的测试在 staging 中通过并不意味着你的系统就能工作。开发与生产环境的一致性，往大了说也是个善意的谎言。[我会在生产环境测试](https://increment.com/testing/i-test-in-production/)。
- 不同环境之间总有差异，可能影响系统的*行为*。CDN 可能错误地设置了某些 cache headers；你依赖的下游服务可能行为不同；某个配置值可能不正确。但如果你能在生产环境运行你的规约来快速捕获这些问题，岂不是很棒？

#### 插入 _不同的_ 驱动来测试系统的其他部分

这种灵活性允许我们在不同抽象层和架构层测试行为，让我们能进行黑盒测试之外更聚焦的测试。
- 例如，你可能有一个网页背后挂着一个 API。为什么不用同一个规约同时测试两者呢？网页可以用无头浏览器，API 可以用 HTTP 调用。
- 把这个想法再推进一步，理想情况下，我们希望**代码对本质复杂度建模**（作为"领域"代码），所以我们也应该能用规约写单元测试。这能让我们快速得到反馈，确保系统中的本质复杂度被建模并正确表现。


### 验收测试为正确的理由而修改

采用这种方式后，你的规约唯一需要修改的原因，就是系统的行为发生变化，这是合理的。

- 如果你的 HTTP API 必须修改，你有一个明显的地方去更新它：驱动。
- 如果你的标记发生变化，同样地，更新对应的驱动。

随着系统增长，你会发现自己在多个测试中复用驱动，这又意味着如果实现细节变化，你只需要更新一个通常很明显的位置。

做得好的话，这种方式能让我们在实现细节上保有灵活性，在规约上保有稳定性。重要的是，它为管理变更提供了简单且明显的结构，这在系统及其团队成长时变得至关重要。

### 验收测试作为软件开发的方法

在我们的演讲中，Riya 和我讨论了验收测试以及它和 BDD 的关系。我们谈到，开始工作时先尝试 _理解你正在解决的问题_ 并把它表达为规约，能帮你聚焦意图，是开始工作的好方式。

我最早是在 GOOS 中接触到这种工作方式的。我曾在博客上总结过这些想法。下面是我那篇 [Why TDD](https://quii.dev/The_Why_of_TDD) 文章的摘录

---

TDD 关注的是让你能精确地、迭代地针对所需的行为进行设计。当你开始一个新领域时，你必须识别出一个关键的、必要的行为，并大刀阔斧地砍掉范围。

遵循"自顶向下"的方式，从一个验收测试（AT）开始，让它从外部驱动行为。这会成为你工作的"北极星"。你应该专注的，就是让这个测试通过。在你写出足够的代码让它通过之前，这个测试很可能会持续失败一段时间。

![](https://i.imgur.com/pxTaYu4.png)

一旦你的验收测试搭好，你就可以进入 TDD 流程，驱动出足够的单元来让验收测试通过。诀窍是不要在这一步过分担心设计；写够代码让验收测试通过即可，因为你还在学习和探索这个问题。

走出这第一步通常比你想象的要繁琐得多——搭建 Web 服务器、路由、配置等等——这正是为什么把工作范围保持得很小至关重要。我们想在空白画布上迈出第一步积极的步伐，并以一个通过的验收测试作为支撑，这样我们就能继续快速且安全地迭代。

![](https://i.imgur.com/t5y5opw.png)

随着你不断开发，倾听你的测试，它们会发出信号帮你把设计推向更好的方向，但同样要锚定在行为上，而不是凭空想象。

通常，第一个为了让验收测试通过而苦干的"单元"会越长越大、变得不舒适，即使是这一点点行为也是如此。这时你就可以开始思考如何拆解问题、引入新的协作者。

![](https://i.imgur.com/UYqd7Cq.png)

这就是测试替身（fakes、mocks 等）派上用场的地方，因为软件内部的大多数复杂度通常不在某个单一实现细节里，而在单元 _之间_、在它们如何交互上。

#### 自底向上的危险

这是"自顶向下"而不是"自底向上"的方式。自底向上有它的用途，但带有一定风险。在没有快速集成进应用、也没有用一个高层测试验证的情况下构建"服务"和代码，**你可能会在尚未验证的想法上浪费大量精力**。

这是验收测试驱动方式的一个关键属性，用测试去对我们的代码做真正的验证。

我太多次见到工程师独立地、自底向上地写出一大块代码，他们以为能解决某个工作，但结果是：

- 没法按我们想要的方式工作
- 做了我们不需要的事
- 不容易集成
- 反正还是要大改

这就是浪费。

## 说够了，开始写代码

和其他章节不同，你需要安装 [Docker](https://www.docker.com)，因为我们要在容器里运行应用。本书读到这里，我们假设你已经能熟练编写 Go 代码、从不同的包里 import 等等。

用 `go mod init github.com/quii/go-specs-greet` 创建一个新项目（这里你可以填任何你喜欢的，但如果改了路径，所有内部 import 都要相应改动）

新建一个 `specifications` 文件夹来存放规约，并加一个文件 `greet.go`

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet() (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet()
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, world")
}
```

我的 IDE（Goland）会帮我自动添加依赖，如果你需要手动添加，可以执行

`go get github.com/alecthomas/assert/v2`

按照 Farley 的验收测试设计（Specification->DSL->Driver->System），现在我们已经把规约和实现解耦了。它不知道、也不在乎我们 _怎么_ `Greet`；它只关心领域里的本质复杂度。诚然，这里的复杂度目前还不多，但我们会在迭代中扩展规约以加入更多功能。从小处开始总是很重要的！

你可以把这个接口看作 DSL 的第一步；随着项目增长，你可能会发现需要不同的抽象方式，但目前这样就挺好。

到这里，为了把规约和实现解耦做这种程度的"仪式"，可能有人会指责我们"过度抽象"。**我向你保证，与实现耦合过紧的验收测试会成为工程团队的真正负担**。我有信心，业界大多数验收测试维护成本高昂的原因，是这种不恰当的耦合，而不是相反——抽象过度。

我们可以用这个规约去验证任何能 `Greet` 的"系统"。

### 第一个系统：HTTP API

我们要通过 HTTP 提供一个"问候服务"。所以我们需要创建：

1. 一个**驱动**。在我们的例子里，它通过使用一个 **HTTP 客户端** 与 HTTP 系统对接。这段代码知道如何与我们的 API 工作。驱动把 DSL 翻译成系统特定的调用；在我们的例子中，驱动会实现规约定义的接口。
2. 一个带有 greet API 的 **HTTP 服务器**
3. 一个**测试**，它负责管理服务器启动和拆除的生命周期，然后把驱动接入规约并把它当作测试运行

## 先写测试

为一个程序创建黑盒测试的初始过程——编译并运行程序、执行测试、最后清理一切——可能相当费力。这就是为什么最好在项目开头、功能最少的时候做这件事。我通常会用一个 "hello world" 服务器实现来开始所有项目，把所有测试都搭好、准备好，让我之后可以快速构建真正的功能。

"规约"、"驱动"和"验收测试"的心智模型可能要花点时间适应，所以请仔细跟着。从尝试调用规约开始"逆向"工作可能会有帮助。

为我们打算交付的程序搭建一些目录结构。

`mkdir -p cmd/httpserver`

在新文件夹里创建一个新文件 `greeter_server_test.go`，加入下面的内容。

```go
package main_test

import (
	"testing"

	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	specifications.GreetSpecification(t, nil)
}
```

我们想在 Go 测试里运行规约。我们已经有 `*testing.T`，所以那是第一个参数，那第二个呢？

`specifications.Greeter` 是个接口，我们要用一个 `Driver` 去实现它，把新的 TestGreeterServer 代码改成下面这样：

```go
import (
	go_specs_greet "github.com/quii/go-specs-greet"
)

func TestGreeterServer(t *testing.T) {
	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

让我们的 `Driver` 可配置，以便在不同环境（包括本地）运行是很有好处的，所以我们加了一个 `BaseURL` 字段。

## 尝试运行测试

```
./greeter_server_test.go:46:12: undefined: go_specs_greet.Driver
```

我们仍然在实践 TDD！这第一步走得很大；我们需要创建好几个文件，写的代码可能比平常要多，但刚开始时通常都是这样。所以非常重要的一点是要记住红色阶段的规则。

> 为了让测试通过，可以"犯任何必要的错"

## 写最少量的代码让测试能跑，并查看失败的测试输出

捏住鼻子吧；记住，一旦测试通过我们就能重构。下面是 `driver.go` 的代码，我们把它放在项目根目录：

```go
package go_specs_greet

import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
}

func (d Driver) Greet() (string, error) {
	res, err := http.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```


注意：

- 你可以争论说我应该写测试驱动出每一个 `if err != nil`，但根据我的经验，只要你对 `err` 不做额外处理，那种"返回你拿到的错误"的测试价值相对较低。
- **不要使用默认的 HTTP 客户端**。稍后我们会传入一个 HTTP 客户端来配置超时等，但现在我们只是想先让测试通过。
-  在 `greeter_server_test.go` 里我们调用了刚刚创建的 `go_specs_greet` 包的 Driver 函数，别忘了把 `github.com/quii/go-specs-greet` 添加到它的 imports 里。
重新运行测试；它们现在应该能编译但不能通过。

```
Get "http://localhost:8080/greet": dial tcp [::1]:8080: connect: connection refused
```

我们有一个 `Driver`，但还没启动我们的应用，所以它发不了 HTTP 请求。我们需要让验收测试协调构建、运行，最后销毁我们的系统，才能让测试跑起来。

### 运行我们的应用

团队为部署构建系统的 Docker 镜像很常见，所以测试中我们也这么做

为了帮我们在测试中使用 Docker，我们将使用 [Testcontainers](https://golang.testcontainers.org)。Testcontainers 给了我们一种以编程方式构建 Docker 镜像和管理容器生命周期的能力。

`go get github.com/testcontainers/testcontainers-go`

现在你可以把 `cmd/httpserver/greeter_server_test.go` 改成下面这样：

```go
package main_test

import (
	"context"
	"testing"

	"github.com/alecthomas/assert/v2"
	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func TestGreeterServer(t *testing.T) {
	ctx := context.Background()

	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "./cmd/httpserver/Dockerfile",
			// 如果你想少看点输出可以设为 false，但出问题时这很有帮助
			PrintBuildLog: true,
		},
		ExposedPorts: []string{"8080:8080"},
		WaitingFor:   wait.ForHTTP("/").WithPort("8080"),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})

	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

试着运行测试。

```
=== RUN   TestGreeterHandler
2022/09/10 18:49:44 Starting container id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Waiting for container id 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Container is ready id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
    greeter_server_test.go:32: Did not expect an error but got:
        Error response from daemon: Cannot locate specified Dockerfile: ./cmd/httpserver/Dockerfile: failed to create container
--- FAIL: TestGreeterHandler (0.59s)
```

我们需要为程序创建一个 Dockerfile。在 `httpserver` 文件夹里创建 `Dockerfile`，添加如下内容。

```dockerfile
# 确保使用与 go.mod 文件中相同的 Go 版本。
# 例如 golang:1.22.1-alpine。
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/httpserver/*.go

EXPOSE 8080
CMD [ "./svr" ]
```

不要太纠结这里的细节；它可以被打磨和优化，但对于这个示例已经够用。我们这个方式的优势在于，我们以后可以改进 Dockerfile，并通过测试证明它如我们所愿地工作。这是黑盒测试的真正优势！

重新运行测试；它会抱怨没法构建镜像。当然了，因为我们还没写要构建的程序！

为了让测试完整执行，我们需要创建一个监听 `8080` 端口的程序，但**仅此而已**。坚持 TDD 的纪律，在测试如我们预期那样失败之前，不要写让测试通过的生产代码。

在 `httpserver` 文件夹中创建一个 `main.go`：

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

再次尝试运行测试，它应该会以下面的方式失败。

```
    greet.go:16: Expected values to be equal:
        +Hello, World
        \ No newline at end of file
--- FAIL: TestGreeterHandler (2.09s)
```

## 写足够的代码让它通过

更新 handler，让它如我们规约所要求的那样行事

```go
import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		fmt.Fprint(w, "Hello, world")
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

## 重构

虽然严格说这不算重构，但我们不应该依赖默认的 HTTP 客户端，所以让我们改一下 Driver，让它可以从外部传入一个客户端，由测试提供。

```go
import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
	Client  *http.Client
}

func (d Driver) Greet() (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

在我们 `cmd/httpserver/greeter_server_test.go` 的测试中，更新 driver 的创建以传入一个客户端。

```go
client := http.Client{
	Timeout: 1 * time.Second,
}

driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080", Client: &client}
specifications.GreetSpecification(t, driver)
```

让 `main.go` 尽量简单是一种好习惯；它应该只关心把你创建的构建块拼装成一个应用。

在项目根目录创建一个名为 `handler.go` 的文件，把代码挪到那里。

```go
package go_specs_greet

import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello, world")
}
```

更新 `main.go` 改为 import 并使用这个 handler。

```go
package main

import (
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet"
)

func main() {
	handler := http.HandlerFunc(go_specs_greet.Handler)
	http.ListenAndServe(":8080", handler)
}
```

## 反思

第一步感觉颇费力。我们创建了好几个 `go` 文件，去创建并测试一个返回硬编码字符串的 HTTP handler。这种"第 0 次迭代"的仪式和搭建工作会在后续迭代中给我们带来回报。

修改功能应该是简单的、由规约驱动的过程，然后处理它强制我们做出的任何变更。现在 `DockerFile` 和 `testcontainers` 已经为验收测试搭好；除非应用的构成方式发生变化，否则我们应该不需要再改这些文件。

我们将通过下一个需求——问候特定的人——来看到这一点。

## 先写测试

修改我们的规约

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet(name string) (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet("Mike")
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, Mike")
}
```

为了能问候具体的人，我们需要修改对系统的接口，使其接收一个 `name` 参数。

## 尝试运行测试

```
./greeter_server_test.go:48:39: cannot use driver (variable of type go_specs_greet.Driver) as type specifications.Greeter in argument to specifications.GreetSpecification:
	go_specs_greet.Driver does not implement specifications.Greeter (wrong type for Greet method)
		have Greet() (string, error)
		want Greet(name string) (string, error)
```

规约的变化意味着我们的 driver 也要更新。

## 写最少量的代码让测试能跑，并查看失败的测试输出

更新 driver，让它在请求中带上一个 `name` 查询参数，请求问候特定的 `name`。

```go
import "io"

func (d Driver) Greet(name string) (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet?name=" + name)
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

测试现在应该能跑起来，并且失败。

```
    greet.go:16: Expected values to be equal:
        -Hello, world
        \ No newline at end of file
        +Hello, Mike
        \ No newline at end of file
--- FAIL: TestGreeterHandler (1.92s)
```

## 写足够的代码让它通过

从请求中提取 `name` 并问候之。

```go
import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %s", r.URL.Query().Get("name"))
}
```

测试现在应该通过。

## 重构

在 [HTTP Handlers Revisited](https://github.com/quii/learn-go-with-tests/blob/main/http-handlers-revisited.md) 中，我们讨论过 HTTP handler 应该只负责处理 HTTP 相关的事情；任何"领域逻辑"都应该放在 handler 之外。这让我们能在脱离 HTTP 的环境下开发领域逻辑，从而更易测试和理解。

让我们把这些关注点拆开。

把 `./handler.go` 中的 handler 更新如下：

```go
func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, Greet(name))
}
```

新建文件 `./greet.go`：
```go
package go_specs_greet

import "fmt"

func Greet(name string) string {
	return fmt.Sprintf("Hello, %s", name)
}
```

## 关于"适配器"设计模式的一点小插曲

既然我们已经把"问候人"的领域逻辑分离到一个独立的函数里，我们就可以直接为 greet 函数写单元测试了。这显然比通过一个走完驱动、再打到 Web 服务器、最后拿到一个字符串的规约去测试简单得多！

如果在这里也能复用我们的规约岂不是很棒？毕竟，规约的要点就是与实现细节解耦。如果规约捕获的是我们的**本质复杂度**，而我们的"领域"代码是用来对它建模的，那我们应该能把它们一起使用。

我们试一下，按下面的样子创建 `./greet_test.go`：

```go
package go_specs_greet_test

import (
	"testing"

	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(t, go_specs_greet.Greet)
}

```

这样很美好，可惜跑不起来

```
./greet_test.go:11:39: cannot use go_specs_greet.Greet (value of type func(name string) string) as type specifications.Greeter in argument to specifications.GreetSpecification:
	func(name string) string does not implement specifications.Greeter (missing Greet method)
```

我们的规约想要一个有 `Greet()` 方法的东西，而不是一个函数。

这个编译错误让人沮丧；我们手里有一个我们"知道"是 `Greeter` 的东西，但它的**形状**还不太对，不能让编译器允许我们使用它。这就是**适配器**模式所要解决的。

> 在[软件工程](https://en.wikipedia.org/wiki/Software_engineering)中，**适配器模式**是一种[软件设计模式](https://en.wikipedia.org/wiki/Software_design_pattern)（也叫做 [wrapper](https://en.wikipedia.org/wiki/Wrapper_function)，是一个与[装饰器模式](https://en.wikipedia.org/wiki/Decorator_pattern)共享的别名），它允许把一个已存在的[类](https://en.wikipedia.org/wiki/Class_(computer_science))的[接口](https://en.wikipedia.org/wiki/Interface_(computer_science))当作另一个接口来使用。[[1\]](https://en.wikipedia.org/wiki/Adapter_pattern#cite_note-HeadFirst-1) 它常被用来在不修改[源代码](https://en.wikipedia.org/wiki/Source_code)的情况下让现有类与其他类协同工作。

设计模式经常如此——一堆华丽的字描述一件相对简单的事，这也是为什么人们倾向于翻白眼。设计模式的价值不在于具体实现，而在于提供一种语言来描述工程师常遇到的常见问题的特定解法。如果你的团队有共享的词汇，沟通的摩擦就会减少。

把下面的代码加到 `./specifications/adapters.go`

```go
type GreetAdapter func(name string) string

func (g GreetAdapter) Greet(name string) (string, error) {
	return g(name), nil
}
```

我们现在可以在测试中使用这个适配器，把我们的 `Greet` 函数接到规约上。

```go
package go_specs_greet_test

import (
	"testing"

	gospecsgreet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(
		t,
		specifications.GreetAdapter(gospecsgreet.Greet),
	)
}
```

当你有一个具备某接口所要求行为，但形状不对的类型时，适配器模式很方便。

## 反思

行为变更感觉很简单，对吧？好吧，也许只是问题本身的特性所致，但这种工作方式给了你纪律，以及一种简单、可重复的方式从上到下改变系统：

- 分析你的问题，识别一个能把系统朝正确方向推进一小步的轻微改进
- 把新的本质复杂度记录在规约中
- 跟着编译错误走，直到验收测试能跑起来
- 更新实现，让系统按照规约的要求工作
- 重构

熬过第一次迭代的痛苦后，我们没必要再改验收测试代码了，因为我们已经分离了规约、驱动和实现。改变规约使我们必须更新驱动，最终更新实现，但围绕 _如何_ 把系统作为容器跑起来的样板代码不受影响。

即使加上为应用构建 Docker 镜像和启动容器的开销，对**整个**应用的测试反馈循环依然非常紧凑：

```
quii@Chriss-MacBook-Pro go-specs-greet % go test ./...
ok  	github.com/quii/go-specs-greet	0.181s
ok  	github.com/quii/go-specs-greet/cmd/httpserver	2.221s
?   	github.com/quii/go-specs-greet/specifications	[no test files]
```

现在，假设你的 CTO 决定 gRPC 是 _未来的方向_。她希望你在保留现有 HTTP 服务器的同时，通过一个 gRPC 服务器对外提供同样的功能。

这是**意外复杂度**的一个例子。记住，意外复杂度是因为我们要和计算机打交道而带来的复杂度，比如网络、磁盘、API 等等。**本质复杂度并未改变**，所以我们不应该需要改规约。

许多仓库结构和设计模式主要是在处理对各种复杂度的分离。例如"端口与适配器"（ports and adapters）要求你把领域代码和任何与意外复杂度相关的事物分开；那种代码放在 "adapters" 文件夹里。

### 让变更变得容易

有时，在做一个变更 _之前_ 先做点重构是有意义的。

> 先让变更变得容易，然后再做容易的变更

~Kent Beck

为此，我们把 `http` 相关代码——`driver.go` 和 `handler.go`——移到 `adapters` 文件夹下的 `httpserver` 包，并把它们的包名改成 `httpserver`。

现在你需要在 `handler.go` 里 import 根包，以引用 Greet 方法……

```go
package httpserver

import (
	"fmt"
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet/domain/interactions"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, go_specs_greet.Greet(name))
}

```

把你的 httpserver 适配器 import 到 main.go 里：

```go
package main

import (
	"net/http"

	"github.com/quii/go-specs-greet/adapters/httpserver"
)

func main() {
	handler := http.HandlerFunc(httpserver.Handler)
	http.ListenAndServe(":8080", handler)
}
```

并更新 greeter_server_test.go 中对 `Driver` 的 import 和引用：

```go
driver := httpserver.Driver{BaseURL: "http://localhost:8080", Client: &client}
```

最后，把领域级代码也归到它自己的文件夹里会很有帮助。别偷懒，别在你的项目里弄一个塞了几百个不相干类型和函数的 `domain` 文件夹。花点心思想想你的领域，把属于一起的概念归到一起。这能让你的项目更易理解，也能改善 import 的质量。

与其看到

```go
domain.Greet
```

——这有点别扭——不如倾向写成

```go
interactions.Greet
```

创建一个 `domain` 文件夹存放所有领域代码，里面再加一个 `interactions` 文件夹。根据你的工具不同，可能要更新一些 import 和代码。

我们的项目树现在应该是这样：

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   └── httpserver
|       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── adapters.go
    └── greet.go

```

我们的领域代码——**本质复杂度**——位于 go module 的根部，而把它们用到"现实世界"中的代码被组织成各种**适配器**。`cmd` 文件夹是我们把这些逻辑分组组合成实际应用的地方，并通过黑盒测试验证一切运转正常。漂亮！

最后，我们可以稍微整理一下验收测试。如果你看一下验收测试的高层步骤：

- 构建 docker 镜像
- 等它在 _某个_ 端口上开始监听
- 创建一个理解如何把 DSL 翻译成系统特定调用的驱动
- 把驱动接到规约中

……你会意识到 gRPC 服务器的验收测试需求是一样的！

`adapters` 文件夹看起来是个不错的位置，所以在一个名为 `docker.go` 的文件中，把前两步封装在一个之后会复用的函数里。

```go
package adapters

import (
	"context"
	"fmt"
	"testing"
	"time"

	"github.com/alecthomas/assert/v2"
	"github.com/docker/go-connections/nat"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func StartDockerServer(
	t testing.TB,
	port string,
	dockerFilePath string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:       "../../.",
			Dockerfile:    dockerFilePath,
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

这样我们就能稍微精简验收测试

```go
func TestGreeterServer(t *testing.T) {
	var (
		port           = "8080"
		dockerFilePath = "./cmd/httpserver/Dockerfile"
		baseURL        = fmt.Sprintf("http://localhost:%s", port)
		driver         = httpserver.Driver{BaseURL: baseURL, Client: &http.Client{
			Timeout: 1 * time.Second,
		}}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, driver)
}
```

这应该会让写 _下一个_ 测试更简单。

## 先写测试

这个新功能可以通过创建一个新的 `adapter` 与领域代码交互来完成。因此我们：

- 不需要改规约；
- 应该能复用规约；
- 应该能复用领域代码。

在 `cmd` 中创建一个新文件夹 `grpcserver` 来存放新程序及其对应的验收测试。在 `cmd/grpc_server/greeter_server_test.go` 中，加入一个验收测试，它和我们 HTTP 服务器的测试看起来非常相似，这不是巧合，而是设计如此。

```go
package main_test

import (
	"fmt"
	"testing"

	"github.com/quii/go-specs-greet/adapters"
	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	var (
		port           = "50051"
		dockerFilePath = "./cmd/grpcserver/Dockerfile"
		driver         = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, &driver)
}
```

唯一的不同点是：

- 我们使用了不同的 docker file，因为我们要构建的是另一个程序
- 这意味着我们需要一个新的 `Driver`，它会用 `gRPC` 与新程序交互

## 尝试运行测试

```
./greeter_server_test.go:26:12: undefined: grpcserver
```

我们还没创建 `Driver`，所以编译不通过。

## 写最少量的代码让测试能跑，并查看失败的测试输出

在 `adapters` 中创建文件夹 `grpcserver`，并在其中创建 `driver.go`

```go
package grpcserver

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	return "", nil
}
```

如果你再运行一次，它现在应该能 _编译_，但不会通过，因为我们还没有创建 Dockerfile 和对应的程序去运行。

在 `cmd/grpcserver` 内创建一个新的 `Dockerfile`。

```dockerfile
# 确保使用与 go.mod 文件中相同的 Go 版本。
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/grpcserver/*.go

EXPOSE 50051
CMD [ "./svr" ]
```

以及一个 `main.go`

```go
package main

import "fmt"

func main() {
	fmt.Println("implement me")
}
```

你现在应该会看到测试因为我们的服务没在端口上监听而失败。是时候开始用 gRPC 构建客户端和服务器了。

## 写足够的代码让它通过

### gRPC

如果你不熟悉 gRPC，建议从 [gRPC 官网](https://grpc.io)看起。不过对本章而言，它只是接入我们系统的另一种适配器，是其他系统调用（**r**emote **p**rocedure **c**all）我们出色领域代码的一种方式。

转折点在于你用 Protocol Buffers 定义一个"服务定义"。然后从这个定义生成服务端和客户端代码。这不仅适用于 Go，也适用于大多数主流语言。这意味着你可以把定义分享给公司里甚至不写 Go 的其他团队，仍然能顺利做服务对服务的通信。

如果你之前没用过 gRPC，你需要安装一个 **Protocol buffer 编译器**和一些 **Go 插件**。[gRPC 官网有清晰的安装说明](https://grpc.io/docs/languages/go/quickstart/)。

在我们新 driver 的同一文件夹里，添加一个 `greet.proto` 文件，内容如下

```protobuf
syntax = "proto3";

option go_package = "github.com/quii/adapters/grpcserver";

package grpcserver;

service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
}

message GreetRequest {
  string name = 1;
}

message GreetReply {
  string message = 1;
}
```

要理解这个定义你不需要是 Protocol Buffers 专家。我们定义了一个带 Greet 方法的服务，然后描述了入参和出参的消息类型。

在 `adapters/grpcserver` 内运行下面的命令来生成客户端和服务端代码

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

如果工作正常，我们就拿到一些自动生成的代码可供使用。我们先在 `Driver` 里使用生成的客户端代码。

```go
package grpcserver

import (
	"context"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	//todo: 我们不应该每次调用 greet 都重新拨号，等绿了之后再重构
	conn, err := grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		return "", err
	}
	defer conn.Close()

	client := NewGreeterClient(conn)
	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

现在我们有了客户端，需要更新 `main.go` 来创建一个服务端。记住，这个阶段我们只是想让测试通过，先不操心代码质量。

```go
package main

import (
	"context"
	"log"
	"net"

	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"google.golang.org/grpc"
)

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatal(err)
	}
	s := grpc.NewServer()
	grpcserver.RegisterGreeterServer(s, &GreetServer{})

	if err := s.Serve(lis); err != nil {
		log.Fatal(err)
	}
}

type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: "fixme"}, nil
}
```

为了创建 gRPC 服务器，我们必须实现它给我们生成的接口

```go
// GreeterServer is the server API for Greeter service.
// All implementations must embed UnimplementedGreeterServer
// for forward compatibility
type GreeterServer interface {
	Greet(context.Context, *GreetRequest) (*GreetReply, error)
	mustEmbedUnimplementedGreeterServer()
}
```

我们的 `main` 函数：

- 监听一个端口
- 创建一个实现了该接口的 `GreetServer`，然后通过 `grpcServer.RegisterGreeterServer` 把它和一个 `grpc.Server` 一起注册。
- 让服务器使用监听器

把消息从硬编码的 `fix-me` 改成调用我们的领域代码并不会多费事，但我想先运行验收测试看看在传输层一切是否正常工作，并验证失败测试输出。

```
greet.go:16: Expected values to be equal:
-fixme
\ No newline at end of file
+Hello, Mike
\ No newline at end of file
```

漂亮！我们能看到测试中我们的驱动能够连上 gRPC 服务器。

现在，在我们的 `GreetServer` 中调用领域代码

```go
type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

终于通过了！我们有了一个验收测试，证明我们的 gRPC greet 服务器按我们的期望工作。

## 重构

为了让测试通过，我们犯了一些"罪"，但既然测试都过了，我们就有了重构的安全网。

### 简化 main

像前面一样，我们不希望 `main` 里有太多代码。可以把新的 `GreetServer` 移到 `adapters/grpcserver`，因为它本来就该住在那。从内聚的角度看，如果服务定义发生变化，我们希望变更的"波及范围"被限制在代码的那个区域内。

### 不要每次都让 driver 重新拨号

我们目前只有一个测试，但如果我们扩展规约（我们会的），让 Driver 在每次 RPC 调用时都重新拨号是没意义的。

```go
package grpcserver

import (
	"context"
	"sync"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string

	connectionOnce sync.Once
	conn           *grpc.ClientConn
	client         GreeterClient
}

func (d *Driver) Greet(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}

func (d *Driver) getClient() (GreeterClient, error) {
	var err error
	d.connectionOnce.Do(func() {
		d.conn, err = grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
		d.client = NewGreeterClient(d.conn)
	})
	return d.client, err
}
```

这里我们演示了如何用 [`sync.Once`](https://pkg.go.dev/sync#Once) 确保 `Driver` 只尝试创建一次到服务端的连接。

在继续之前，我们看一下当前项目结构。

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   ├── docker.go
│   ├── grpcserver
│   │   ├── driver.go
│   │   ├── greet.pb.go
│   │   ├── greet.proto
│   │   ├── greet_grpc.pb.go
│   │   └── server.go
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   ├── grpcserver
│   │   ├── Dockerfile
│   │   ├── greeter_server_test.go
│   │   └── main.go
│   └── httpserver
│       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── greet.go
```

- `adapters` 把内聚的功能单元分组放在一起
- `cmd` 存放我们的应用及对应的验收测试
- 我们的代码与所有意外复杂度完全解耦

### 整合 `Dockerfile`

你大概注意到两个 `Dockerfile` 几乎相同，除了我们要构建的二进制路径。

`Dockerfile` 可以接受参数让我们在不同上下文中复用它，听起来正合适。我们可以删掉两个 Dockerfile，改为在项目根目录放一个：

```dockerfile
# 确保使用与 go.mod 文件中相同的 Go 版本。
FROM golang:1.18-alpine

WORKDIR /app

ARG bin_to_build

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/${bin_to_build}/main.go

CMD [ "./svr" ]
```

在构建镜像时，我们要更新 `StartDockerServer` 把这个参数传进去

```go
func StartDockerServer(
	t testing.TB,
	port string,
	binToBuild string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "Dockerfile",
			BuildArgs: map[string]*string{
				"bin_to_build": &binToBuild,
			},
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

最后，更新我们的测试以传入要构建的镜像（另一个测试也一样改，把 `grpcserver` 换成 `httpserver`）。

```go
func TestGreeterServer(t *testing.T) {
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
}
```

### 区分不同种类的测试

验收测试的优点是从纯用户视角、行为视角测试整个系统是否正常工作，但相比单元测试它们也有缺点：

- 慢
- 反馈质量通常不如单元测试聚焦
- 对内部质量或设计帮助不大

[The Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) 会指引我们如何为测试套件搭配比例，建议你阅读 Fowler 的原文了解更多细节，而本文最简化的总结就是"大量单元测试 + 少量验收测试"。

因此，随着项目变大，你可能会遇到验收测试需要几分钟才能跑完的情况。为了给查看你项目的人提供更友好的开发者体验，你可以让开发者分别运行不同类型的测试。

我们偏好的是，工程师无需做太多搭建（除了几个关键依赖比如 Go 编译器（显然）和也许 Docker），就能直接 `go test ./...` 跑起来。

Go 给工程师提供了一种只跑"短"测试的机制——[short flag](https://pkg.go.dev/testing#Short)

`go test -short ./...`

我们可以在验收测试里检查 flag 的值，看用户是不是想运行验收测试

```go
if testing.Short() {
	t.Skip()
}
```

我做了一个 `Makefile` 来展示这种用法

```makefile
build:
	golangci-lint run
	go test ./...

unit-tests:
	go test -short ./...
```

### 我应该什么时候写验收测试？

最佳实践是倾向于大量快速的单元测试加少量的验收测试，但你怎么决定该写验收测试还是单元测试呢？

很难给出具体规则，但我通常会问自己以下几个问题：

- 这是个边界情况吗？我更倾向于用单元测试覆盖
- 这是非技术人员经常谈论的东西吗？那么我希望对关键功能的"真正能用"有很高的信心，会加一个验收测试
- 我描述的是用户旅程，而不是某个具体函数吗？验收测试
- 单元测试给我足够信心了吗？有时你是在已有验收测试的旅程上加新功能，处理不同的输入下不同的场景。这种情况下加一个验收测试成本高、价值低，我更倾向写单元测试。

## 在我们的工作上迭代

经过这么多努力，你应该会希望扩展系统现在变得简单。让一个系统易于扩展并不一定容易，但这值得花时间，并且当你在项目早期就开始这么做时，会比之后再做容易得多。

让我们扩展 API，加一个"诅咒"功能。

## 先写测试

这是一个全新的行为，所以我们应该从验收测试开始。在我们的规约文件中，加上下面的内容

```go
type MeanGreeter interface {
	Curse(name string) (string, error)
}

func CurseSpecification(t *testing.T, meany MeanGreeter) {
	got, err := meany.Curse("Chris")
	assert.NoError(t, err)
	assert.Equal(t, got, "Go to hell, Chris!")
}
```

挑一个我们的验收测试，试着使用这个规约

```go
func TestGreeterServer(t *testing.T) {
	if testing.Short() {
		t.Skip()
	}
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	t.Cleanup(driver.Close)
	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
	specifications.CurseSpecification(t, &driver)
}
```

## 尝试运行测试

```
# github.com/quii/go-specs-greet/cmd/grpcserver_test [github.com/quii/go-specs-greet/cmd/grpcserver.test]
./greeter_server_test.go:27:39: cannot use &driver (value of type *grpcserver.Driver) as type specifications.MeanGreeter in argument to specifications.CurseSpecification:
	*grpcserver.Driver does not implement specifications.MeanGreeter (missing Curse method)
```

我们的 `Driver` 还不支持 `Curse`。

## 写最少量的代码让测试能跑，并查看失败的测试输出

记住我们只是想让测试能跑，所以给 `Driver` 加上方法

```go
func (d *Driver) Curse(name string) (string, error) {
	return "", nil
}
```

如果你再试一次，测试应该能编译、能运行、并失败

```
greet.go:26: Expected values to be equal:
+Go to hell, Chris!
\ No newline at end of file
```

## 写足够的代码让它通过

我们需要更新 protocol buffer 规约，让它带上 `Curse` 方法，然后重新生成代码。

```protobuf
service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
  rpc Curse (GreetRequest) returns (GreetReply) {}
}
```

你可以争论说复用 `GreetRequest` 和 `GreetReply` 类型是不恰当的耦合，但我们可以在重构阶段处理它。我一直强调，我们只是要让测试通过，先验证软件能用，_然后_ 再让它变好看。

用（在 `adapters/grpcserver` 内）这条命令重新生成代码。

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

### 更新 driver

客户端代码更新之后，我们就可以在 `Driver` 中调用 `Curse`

```go
func (d *Driver) Curse(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Curse(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

### 更新 server

最后，我们要在 `Server` 上加 `Curse` 方法

```go
package grpcserver

import (
	"context"
	"fmt"

	"github.com/quii/go-specs-greet/domain/interactions"
)

type GreetServer struct {
	UnimplementedGreeterServer
}

func (g GreetServer) Curse(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: fmt.Sprintf("Go to hell, %s!", request.Name)}, nil
}

func (g GreetServer) Greet(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

测试现在应该通过。

## 重构

试着自己来做这一步。

- 把 `Curse` 的"领域逻辑"从 grpc server 中抽离出来，就像我们之前对 `Greet` 做的那样。把规约用作针对你领域逻辑的单元测试
- 在 protobuf 中使用不同的类型，让 `Greet` 和 `Curse` 的消息类型解耦。

## 为 HTTP 服务器实现 `Curse`

同样，留作读者的练习。我们已经有了领域级规约和领域级逻辑，并且整齐地分离了。如果你跟着本章走过来了，这应该非常直接。

- 把规约加到现有 HTTP 服务器的验收测试中
- 更新你的 `Driver`
- 在服务器上加新端点，并复用领域代码实现功能。你可能想用 `http.NewServeMux` 来处理到不同端点的路由。

记得小步前进，频繁提交并跑测试。如果你真的卡住了，[可以在 GitHub 上查看我的实现](https://github.com/quii/go-specs-greet)。

## 通过单元测试更新领域逻辑来增强两个系统

如前所述，并非系统的每一个变更都要由验收测试驱动。如果你把关注点分离得很好，业务规则的各种排列组合和边界情况，应该能用单元测试很简单地驱动出来。

给我们的 `Greet` 函数加一个单元测试，让 `name` 在为空时默认为 `World`。你应该能感觉到这有多简单，然后两个应用都能"免费"获得这个业务规则。

## 总结

构建变更成本合理的系统，需要你设计验收测试来帮助你，而不是变成维护负担。它们可以作为指引，或者像 GOOS 所说的，方法性地"成长"你的软件。

希望通过这个例子，你能看到我们应用中可预测、有结构的变更工作流，以及你如何在自己工作中应用它。

你可以想象自己和某个利益相关者交谈，他想以某种方式扩展你正在做的系统。在一个面向领域的、与实现无关的规约中捕获它，把它当作你工作的北极星。Riya 和我在 [GopherconUK 演讲](https://www.youtube.com/watch?v=ZMWJCk_0WrY) 中描述了利用 BDD 技巧（如"Example Mapping"）来帮助你更深刻理解本质复杂度，从而能写出更详细、更有意义的规约。

把本质复杂度和意外复杂度的关注点分开，会让你的工作不再是临时拼凑，而是更结构化、更深思熟虑；这能确保验收测试的韧性，让它们少成为维护负担。

Dave Farley 给出了一个绝佳的小贴士：

> 想象一个你能想到的最不懂技术、但理解问题领域的人，去阅读你的验收测试。这些测试应该让那个人也能看懂。

规约这样一来还能兼作文档。它们应该清晰地规定一个系统应该如何行事。这种思想正是 [Cucumber](https://cucumber.io) 这类工具背后的原理，它给你一种 DSL 把行为描述成代码，然后你把这种 DSL 转换成系统调用，就像我们这里做的一样。

### 涵盖了哪些内容

- 写抽象规约能让你表达所解决问题的本质复杂度，并去除意外复杂度。这能让你在不同上下文中复用规约。
- 如何使用 [Testcontainers](https://golang.testcontainers.org) 管理验收测试中系统的生命周期。这让你能在自己电脑上彻底测试要发布的镜像，给你快速反馈和信心。
- 用 Docker 容器化应用的简短入门
- gRPC
- 与其追求现成的目录结构，你可以让开发方式自然地驱动出应用结构，基于你自己的需要。

### 拓展资料

- 在我们的例子里，我们的"DSL"其实算不上什么 DSL；我们只是用接口把规约和现实世界解耦，并允许我们干净地表达领域逻辑。随着系统增长，这个抽象层级可能会变得笨拙、不清晰。如果你想找更多关于如何组织规约的想法，[读读 "Screenplay Pattern"](https://cucumber.io/blog/bdd/understanding-screenplay-(part-1)/)。
- 再强调一下，[Growing Object-Oriented Software, Guided by Tests](http://www.growing-object-oriented-software.com) 是经典之作。它演示了应用这种"伦敦学派"、"自顶向下"方式写软件的过程。任何喜欢《Learn Go with Tests》的人都能从读 GOOS 中获益良多。
- [示例代码仓库](https://github.com/quii/go-specs-greet)中还有更多代码和点子是我没在这里写到的，比如多阶段 docker 构建，你可以去看一下。
  - 特别地，*为了好玩*，我做了一个**第三个程序**：一个带 HTML 表单的网站可以 `Greet` 和 `Curse`。`Driver` 利用了出色的 [https://github.com/go-rod/rod](https://github.com/go-rod/rod) 模块，使它能像用户一样用浏览器与网站交互。看 git 历史就能看到我一开始没用任何模板工具，"先让它跑起来"，等通过验收测试后，我就有了在不担心破坏功能的前提下重构的自由。 -->
