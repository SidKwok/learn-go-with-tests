# Learn Go with Tests

<p align="center">
  <img src="red-green-blue-gophers-smaller.png" />
</p>

[Art by Denise](https://twitter.com/deniseyu21)

[![Go Report Card](https://goreportcard.com/badge/github.com/quii/learn-go-with-tests)](https://goreportcard.com/report/github.com/quii/learn-go-with-tests)

## 格式

- [Gitbook](https://quii.gitbook.io/learn-go-with-tests)
- [EPUB 或 PDF](https://github.com/quii/learn-go-with-tests/releases)

## 翻译版本

- [中文](https://studygolang.gitbook.io/learn-go-with-tests)
- [Português](https://larien.gitbook.io/aprenda-go-com-testes/)
- [日本語](https://andmorefine.gitbook.io/learn-go-with-tests/)
- [Français](https://goosegeesejeez.gitbook.io/apprendre-go-par-les-tests)
- [한국어](https://miryang.gitbook.io/learn-go-with-tests/)
- [Türkçe](https://halilkocaoz.gitbook.io/go-programlama-dilini-ogren/)
- [فارسی](https://go-yaad-begir.gitbook.io/go-ba-test/)
- [Nederlands](https://bobkosse.gitbook.io/leer-go-met-tests/)
- [Tiếng Việt](https://sons-organization-15.gitbook.io/learn-go-with-tests)

## 支持作者

我很高兴能免费提供这份资源，但如果你想表达一些谢意：

- [在 Twitter 上 @quii](https://twitter.com/quii)
- <a rel="me" href="https://mastodon.cloud/@quii">Mastodon</a>
- [请我喝杯咖啡 :coffee:](https://www.buymeacoffee.com/quii)
- [在 GitHub 上赞助我](https://github.com/sponsors/quii)

## 为什么写这本书

* 通过编写测试来探索 Go 语言
* **打好 TDD 的基础**。Go 是学习 TDD 的好语言，因为它语法简单易学，而且测试是内置的
* 让你有信心开始用 Go 编写健壮的、有良好测试覆盖的系统
* [看一段视频，或阅读为什么单元测试和 TDD 很重要](why.md)

## 目录

### Go 基础

1. [安装 Go](install-go.md) - 搭建高效率的开发环境。
2. [Hello, world](hello-world.md) - 声明变量和常量、if/else 语句、switch、写出你的第一个 Go 程序和第一个测试。子测试语法和闭包。
3. [整数](integers.md) - 进一步探索函数声明语法，学习改善代码文档的新方式。
4. [迭代](iteration.md) - 学习 `for` 和基准测试。
5. [数组与切片](arrays-and-slices.md) - 学习数组、切片、`len`、变长参数、`range` 和测试覆盖率。
6. [结构体、方法与接口](structs-methods-and-interfaces.md) - 学习 `struct`、方法、`interface` 和表驱动测试。
7. [指针与错误](pointers-and-errors.md) - 学习指针和错误。
8. [Maps](maps.md) - 学习把值存到 map 数据结构里。
9. [依赖注入](dependency-injection.md) - 学习依赖注入，它与使用接口的关系，以及对 io 的初步介绍。
10. [Mocking](mocking.md) - 拿一些已有的、没有测试的代码，使用 DI 配合 mock 来对它进行测试。
11. [并发](concurrency.md) - 学习如何编写并发代码，让你的软件更快。
12. [Select](select.md) - 学习如何优雅地同步异步流程。
13. [反射](reflection.md) - 学习反射
14. [Sync](sync.md) - 学习 sync 包中的一些功能，包括 `WaitGroup` 和 `Mutex`
15. [Context](context.md) - 使用 context 包来管理和取消长时间运行的流程
16. [基于属性的测试入门](roman-numerals.md) - 通过罗马数字 kata 练习一些 TDD，并简要介绍基于属性的测试
17. [数学](math.md) - 使用 `math` 包绘制一个 SVG 时钟
18. [读取文件](reading-files.md) - 读取文件并处理它们
19. [模板](html-templates.md) - 使用 Go 的 html/template 包从数据渲染出 html，并学习审批测试（approval testing）
20. [泛型](generics.md) - 学习如何编写接收泛型参数的函数，并构建你自己的泛型数据结构
21. [用泛型重新审视数组与切片](revisiting-arrays-and-slices-with-generics.md) - 泛型在处理集合时非常有用。学习如何编写你自己的 `Reduce` 函数，并整理一些常见的模式。

### 构建一个应用

希望你已经消化了 _Go 基础_ 部分，对 Go 大多数语言特性以及如何做 TDD 都有了扎实的基础。

接下来这部分会涉及构建一个应用。

每一章都会在前一章的基础上迭代，按照我们产品负责人的要求扩展应用的功能。

我们会引入新概念来帮助写出优秀的代码，但大部分新内容是学习如何用 Go 标准库去完成任务。

到本章结束时，你应该能比较扎实地掌握如何在测试支持下迭代式地用 Go 编写一个应用。

* [HTTP server](http-server.md) - 我们将创建一个监听 HTTP 请求并响应的应用。
* [JSON、路由与嵌入](json.md) - 我们将让端点返回 JSON 并探索如何做路由。
* [IO 与排序](io.md) - 我们将把数据持久化并从磁盘读取，还会涵盖数据排序。
* [命令行与项目结构](command-line.md) - 在同一份代码库中支持多个应用，并从命令行读取输入。
* [Time](time.md) - 使用 `time` 包来调度活动。
* [WebSockets](websockets.md) - 学习如何编写并测试一个使用 WebSockets 的服务器。

### 测试基础

涵盖测试相关的其他主题。

* [验收测试入门](intro-to-acceptance-tests.md) - 学习如何为你的代码编写验收测试，并通过一个真实例子展示如何优雅地关闭一个 HTTP 服务器
* [扩展验收测试](scaling-acceptance-tests.md) - 学习一些技巧来管理为复杂系统编写验收测试时的复杂度。
* [不使用 mock、stub 和 spy 的测试](working-without-mocks.md) - 学习如何使用 fake 与契约（contract）来创建更真实、更易维护的测试。
* [重构清单](refactoring-checklist.md) - 讨论一下重构是什么，以及一些基本的重构技巧。

### 问与答

我经常在网上遇到这样的问题：

> 我那个做了 x、y、z 的牛逼函数该怎么测？

如果你有这样的问题，可以在 github 上提个 issue，我会尽量抽时间写一篇简短的章节来解决这个问题。我觉得这种内容很有价值，因为它解决的是人们在测试方面的 _真实_ 疑问。

* [OS exec](os-exec.md) - 一个例子，展示我们如何借助操作系统执行命令来获取数据，同时保持业务逻辑可测。
* [错误类型](error-types.md) - 创建你自己的错误类型，以改善你的测试，让你的代码更易使用。
* [感知 context 的 Reader](context-aware-reader.md) - 学习如何用 TDD 给 `io.Reader` 增加取消能力。基于 [Context-aware io.Reader for Go](https://pace.dev/blog/2020/02/03/context-aware-ioreader-for-golang-by-mat-ryer)
* [重新审视 HTTP Handlers](http-handlers-revisited.md) - 测试 HTTP handler 似乎让许多开发者倍感痛苦。本章探讨了正确设计 handler 的相关问题。

### 杂谈 / 讨论

* [为什么要做单元测试，以及如何让它真的为你所用](why.md) - 看一段视频，或阅读为什么单元测试和 TDD 很重要
* [反模式](anti-patterns.md) - 一篇关于 TDD 和单元测试反模式的简短章节

## 贡献

* _这是一个进行中的项目_ 如果你想贡献，请联系我。
* 阅读 [contributing.md](contributing.md) 了解贡献指南
* 有什么想法？开个 issue

## 背景

我有一些向开发团队介绍 Go 的经验，也尝试过不同的方式，去把一群对 Go 感到好奇的人培养成高效编写 Go 系统的团队。

### 哪些方法没有效果

#### 读 _那本_ 书

我们尝试过的一种方式是拿起 [蓝皮书](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440)，每周讨论下一章以及相应的练习。

我很喜欢这本书，但它需要很高的投入度。这本书在解释概念上非常详细，这显然很好，但意味着进度很缓慢——这并不适合所有人。

我发现，虽然有一小部分人会读完第 X 章并完成练习，但很多人没有。

#### 解决一些问题

Kata 很有意思，但它们在学习一门语言时通常涉及面有限；你不太可能用 goroutine 去解一个 kata。

另一个问题是，当大家热情程度不一样时，有些人会比其他人学得更深，他们演示自己做的东西时往往会用到别人不熟悉的特性，把人搞糊涂。

这会让整个学习过程感觉很 _无序_、_零散_。

### 哪种方法有效

到目前为止最有效的方式，是通过阅读 [go by example](https://gobyexample.com/) 慢慢介绍语言的基础，用例子去探索它们，并以小组的形式进行讨论。这比"回家读第 X 章"要更具互动性。

久而久之，团队就在语言的 _语法_ 上打下了坚实的基础，于是我们可以开始构建系统了。

这对我来说就好像学吉他时反复练音阶一样。

不管你觉得自己有多艺术，如果不理解基础并多练习基本功，你不太可能写出好作品。

### 对我而言什么有效

当 _我_ 学一门新的编程语言时，通常会先在 REPL 里折腾一下，但最终我需要更多的结构。

我喜欢的做法是先探索概念，再用测试把这些想法固化下来。测试既能验证我写的代码是否正确，也能记录我学到的特性。

结合我和小组学习的经验，以及我个人的方式，我打算尝试做出一些希望能对其他团队有帮助的东西。通过编写小测试来学习基础，然后你就可以发挥你已有的软件设计技能，交付一些很棒的系统。

## 这本书适合谁

* 对学 Go 感兴趣的人。
* 已经了解一些 Go，但想用 TDD 探索测试的人。

## 你需要准备什么

* 一台电脑！
* [安装好 Go](https://golang.org/)
* 一个文本编辑器
* 一些编程经验。理解 `if`、变量、函数等概念。
* 能熟练使用终端

## 反馈

* 在 [这里](https://github.com/quii/learn-go-with-tests) 提 issue 或 PR，或者 [在 Twitter 上 @quii](https://twitter.com/quii)

[MIT 许可证](LICENSE.md)

[Logo 由 egonelbre 制作](https://github.com/egonelbre) 真厉害！
