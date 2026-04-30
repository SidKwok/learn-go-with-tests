# 通过测试学 Go

![](.gitbook/assets/red-green-blue-gophers-smaller.png)

[Art by Denise](https://twitter.com/deniseyu21)

## 支持作者

我很高兴免费提供这份资源，如果你想表达一点感谢

* [Tweet me @quii](https://twitter.com/quii)
* [Mastodon](https://mastodon.cloud/@quii)
* [Buy me a coffee](https://www.buymeacoffee.com/quii)
* [Sponsor me on GitHub](https://github.com/sponsors/quii)

## 用 Go 学习测试驱动开发

* 通过编写测试来探索 Go 语言
* **打好 TDD 的基础**。Go 是学习 TDD 的一门好语言，因为它本身简单易学，并且测试是内置功能
* 让你有信心开始用 Go 编写健壮的、有充分测试的系统

其他语言版本：

* [中文](https://studygolang.gitbook.io/learn-go-with-tests)
* [Português](https://larien.gitbook.io/aprenda-go-com-testes/)
* [日本語](https://andmorefine.gitbook.io/learn-go-with-tests/)
* [Français](https://goosegeesejeez.gitbook.io/apprendre-go-par-les-tests)
* [한국어](https://miryang.gitbook.io/learn-go-with-tests/)
* [Türkçe](https://halilkocaoz.gitbook.io/go-programlama-dilini-ogren/)
* [Nederlands](https://bobkosse.gitbook.io/leer-go-met-tests)

## 背景

我有一些向开发团队引入 Go 的经验，并尝试过不同的方式，把一群对 Go 感到好奇的人逐步培养成能高效编写 Go 系统的开发者。

### 行不通的做法

#### 读 _那本_ 书

我们尝试过的一种方式是拿起 [蓝皮书](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440)，每周讨论下一章和对应的练习。

我喜欢这本书，但它需要很高的投入度。这本书在概念解释上非常详尽，这显然很棒，但也意味着进度缓慢而稳定——并不是每个人都适合。

我发现虽然有少数人会读完第 X 章并完成练习，但很多人不会。

#### 解决一些问题

代码 kata 很有趣，但它们的学习范围通常有限；你不太可能用 goroutine 去解一个 kata。

另一个问题是当大家热情程度参差不齐的时候。有些人就是会比其他人学得多得多，演示成果时反而会用到别人不熟悉的特性，把其他人弄糊涂。

这最终会让学习体验显得相当 _无章可循_、_随心所欲_。

### 行得通的做法

到目前为止，最有效的方式是通过阅读 [go by example](https://gobyexample.com/) 慢慢介绍语言的基础知识，结合示例去探索，并以小组形式讨论。这是比"回家读第 X 章"更具互动性的方法。

随着时间推移，团队对语言的 _语法_ 打下了扎实的基础，于是我们可以开始构建系统了。

在我看来这就像学吉他时练音阶。

不管你觉得自己有多艺术家气质，如果不理解基础并练习基本功，就很难写出好的音乐。

### 适合我的方式

当 _我_ 学习一门新的编程语言时，我通常会先在 REPL 里随便折腾，但最终我需要更多的结构。

我喜欢做的事情是先探索概念，然后用测试去固化这些想法。测试可以验证我写的代码是正确的，并把我学到的功能记录下来。

结合我和小组一起学习的经验、以及我个人的方式，我打算尝试创造一些希望对其他团队也有用的东西。通过编写小型测试来学习基础，然后你就能运用已有的软件设计技能交付出色的系统。

## 适合谁

* 想入门 Go 的人
* 已经懂一点 Go，但想更深入探索测试的人

## 你需要什么

* 一台电脑！
* [已安装的 Go](https://golang.org/)
* 一个文本编辑器
* 一些编程经验，理解 `if`、变量、函数等概念
* 能熟练使用终端

## 反馈

* 在 [这里](https://github.com/quii/learn-go-with-tests) 提 issue/PR 或者 [tweet me @quii](https://twitter.com/quii)

[MIT 许可证](LICENSE.md)
