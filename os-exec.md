# OS Exec

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/os-exec)**

[keith6014](https://www.reddit.com/user/keith6014) 在 [reddit](https://www.reddit.com/r/golang/comments/aaz8ji/testdata_and_function_setup_help/) 上提问

> 我用 os/exec.Command() 执行了一个命令，它生成了一些 XML 数据。这个命令会在一个叫 GetData() 的函数里执行。

> 为了测试 GetData()，我准备了一些 testdata。

> 在我的 _test.go 里有一个 TestGetData，它会调用 GetData()，但 GetData 内部会用到 os.exec，我希望让它改用我的 testdata。

> 有什么好办法做到？调用 GetData 时是不是该加一个"test"标志参数，让它去读文件，比如写成 GetData(mode string)？

几点想法

- 当某个东西难以测试时，往往是因为关注点没分开
- 不要把"测试模式"加到代码里，而应该用[依赖注入](./dependency-injection.md)，对依赖进行建模并分离关注点。

我斗胆猜测了一下代码可能长什么样

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData() string {
	cmd := exec.Command("cat", "msg.xml")

	out, _ := cmd.StdoutPipe()
	var payload Payload
	decoder := xml.NewDecoder(out)

	// 这 3 个调用都可能返回错误，但为了简洁我忽略了
	cmd.Start()
	decoder.Decode(&payload)
	cmd.Wait()

	return strings.ToUpper(payload.Message)
}
```

- 它使用 `exec.Command`，可以在进程外执行一个外部命令
- 我们用 `cmd.StdoutPipe` 捕获输出，它返回一个 `io.ReadCloser`（这一点稍后会很重要）
- 剩下的代码大致是从[官方优秀文档](https://golang.org/pkg/os/exec/#example_Cmd_StdoutPipe)里复制粘贴过来的。
    - 我们把 stdout 的输出捕获到一个 `io.ReadCloser`，然后 `Start` 命令，再调用 `Wait` 等所有数据被读完。在这两个调用之间，我们把数据解码到我们的 `Payload` 结构体里。

下面是 `msg.xml` 的内容

```xml
<payload>
    <message>Happy New Year!</message>
</payload>
```

我写了一个简单的测试来演示它的运作

```go
func TestGetData(t *testing.T) {
	got := GetData()
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

## 可测试的代码

可测试的代码是解耦的、单一职责的。在我看来，这段代码主要有两个关注点

1. 获取原始的 XML 数据
2. 解码 XML 数据并应用我们的业务逻辑（这里就是对 `<message>` 调用 `strings.ToUpper`）

第一部分只是从标准库示例里抄下来的。

第二部分才是我们的业务逻辑所在，看代码就能发现逻辑里"接缝"从哪里开始：就是我们拿到 `io.ReadCloser` 的地方。我们可以利用这个已有的抽象来分离关注点，让代码可测试。

**`GetData` 的问题在于业务逻辑与获取 XML 的方式耦合在一起了。要让设计更好，我们需要把它们解耦**

我们的 `TestGetData` 可以作为这两个关注点之间的集成测试，所以我们会保留它，确保整体仍然能正常工作。

下面是新分离后的代码

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData(data io.Reader) string {
	var payload Payload
	xml.NewDecoder(data).Decode(&payload)
	return strings.ToUpper(payload.Message)
}

func getXMLFromCommand() io.Reader {
	cmd := exec.Command("cat", "msg.xml")
	out, _ := cmd.StdoutPipe()

	cmd.Start()
	data, _ := io.ReadAll(out)
	cmd.Wait()

	return bytes.NewReader(data)
}

func TestGetDataIntegration(t *testing.T) {
	got := GetData(getXMLFromCommand())
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

现在 `GetData` 只从一个 `io.Reader` 取输入，我们让它变得可测试，并且不再关心数据是怎么来的；任何返回 `io.Reader` 的东西（这非常常见）都可以复用这个函数。比如我们可以改成从 URL 而不是命令行获取 XML。

```go
func TestGetData(t *testing.T) {
	input := strings.NewReader(`
<payload>
    <message>Cats are the best animal</message>
</payload>`)

	got := GetData(input)
	want := "CATS ARE THE BEST ANIMAL"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

```

这是 `GetData` 的一个单元测试示例。

通过分离关注点并利用 Go 已有的抽象，测试我们重要的业务逻辑变得轻而易举。
