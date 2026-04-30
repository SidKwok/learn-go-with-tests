# 重构步骤，入门清单

重构是一项技能，一旦你练习得足够多，在大多数情况下就会变成相当容易、近乎本能的事情。

这个活动经常被与更重大的设计变更混为一谈，但它们是分开的。区分重构和其他编程活动是有帮助的，因为这能让我以清晰和纪律来工作。

## 重构 vs 其他活动

重构只是改进既有代码，<u>不改变行为</u>；因此，测试不应该需要改动。

这就是为什么它是 TDD 循环的第 3 步。一旦你添加了一个行为以及支撑它的测试，重构就应该是一项不需要改动测试代码的活动。**如果你在"重构"代码的同时还需要改测试，你做的是别的事情**。

许多很有用的重构都很容易学、容易做（你的 IDE 几乎完全自动化了其中很多），但随着时间推移，会对我们系统的质量产生巨大影响。

### 其他活动，比如"大"设计

> 那么我没有改"真正的"行为，但我必须改测试？这是什么？

假设你正在处理一个类型，想要改进它代码的质量。*重构不应当要求你改测试*，所以你不能：

- 改变行为
- 改变方法签名

……因为你的测试与这两件事耦合在一起，但你可以：

- 引入私有方法、字段，甚至新的类型和接口
- 改变公有方法的内部实现

如果你想改变某个方法的签名怎么办？

```go
func (b BirthdayGreeter) WishHappyBirthday(age int, firstname, lastname string, email Email) {
	// some fascinating emailing code
}
```

你可能觉得它的参数列表太长了，想给代码带来更多的内聚性和意义。

```go
func (b BirthdayGreeter) WishHappyBirthday(person Person)
```

那么，你现在是在 **设计**，必须确保你小心地推进。如果不带纪律地做这件事，你会把代码、它背后的测试，*以及* 可能依赖它的东西搞乱——记住，使用 `WishHappyBirthday` 的可不只是你的测试。希望它也被"真实"代码使用！

**你应该仍然能用先写测试的方式驱动这个变更**。你可以纠结这究竟算不算"行为"变更，但你想让方法表现得不同。

由于这是一个行为变更，这里也应用 TDD 流程。TDD 的一个好处是，它给了你一种简单、安全、可重复的方式来驱动系统中的行为变更；为什么仅仅因为这件事 *感觉* 不一样就放弃它？

在这种情况下，你会改动你已有的测试以使用新的类型。你通常用来降低风险并带来纪律和清晰度的、TDD 的迭代式小步走，在这些情况下也能帮到你。

很有可能你有几个测试调用 `WishHappyBirthday`；在这种场景下，我建议把除一个之外的所有测试都注释掉，驱动出这次变更，然后逐个处理其余测试。

### 大设计

设计可能需要更重大的变更和更广泛的讨论，并且通常带有一定的主观性。改变系统的部分设计通常是一个比重构更长的过程；尽管如此，你仍然应该努力通过思考如何小步推进来降低风险。

### 见树不见林

> [如果在英式英语中有人 **see the wood for the trees**（在美式英语中是 see the forest for the trees），则代表他们陷于细节，无法注意到事物整体上重要的方面。](https://www.collinsdictionary.com/dictionary/english/cant-see-the-wood-for-the-trees)

当 **底层代码良好分解** 时，谈论"大"的设计问题更容易。如果你和同事每次打开文件都要花大量时间在脑海里解析一团乱麻的代码，你又有多大可能去思考代码的设计呢？

这就是为什么 **持续重构在 TDD 流程中如此重要**。如果我们没能解决小的设计问题，我们就难以工程化更大系统的整体设计。

可悲的是，分解不良的代码会随着工程师在不稳的地基上堆砌复杂度而呈指数级变糟。

## 入门心智清单

**养成在每个 TDD 循环中过一遍心智清单的习惯。** 你越强迫自己去练习，就越容易。**这是一项需要练习的技能。** 记住，这些每一项变更都不应该需要改动你的测试。

我列出了我和同事使用的 IntelliJ/GoLand 的快捷键。每当我指导新工程师时，都会鼓励他们尝试养成肌肉记忆和习惯，使用这些工具来快速、安全地重构。

### 内联变量

如果你创建一个变量只是为了把它传给另一个方法/函数：

```go
url := baseURL + "/user/" + id
res, err := client.Get(url)
```

考虑把它内联（`command+option+n`）*除非* 这个变量名带来了显著的意义。

```go
res, err := client.Get(baseURL + "/user/" + id)
```

不要在内联上 _太_ 聪明；目标不是让变量数为零，从而搞出谁也读不懂的离谱单行代码。如果你能给一个值添加显著的命名，最好就让它保留。

### 用提取变量来 DRY 化值

"不要重复你自己"（DRY）。在一个函数里多次使用同一个值？考虑提取并把它装进一个有意义的变量名（`command+option+v`）。

这有助于可读性，并让将来修改值更容易，因为你不必记得去更新同一值的多个出现位置。

### 一般地 DRY 化

如今 [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) 名声不太好，这有它的合理性。DRY 是那种 _太_ 容易在表面层次上理解、然后被误用的概念之一。

工程师很容易把 DRY 推得太远，为了省几行代码而创造令人费解、纠缠不清的抽象，而不是 DRY _真正_ 的想法——把一个 _概念_ 捕获到一处。减少代码行数往往是 DRY 的一个副产品，**但它不是真正的目标**。

所以是的，DRY 可能被误用，但走到另一个极端，拒绝 DRY 化任何东西也是邪恶的。重复的代码增加了噪音并增加了维护成本。出于对滥用 DRY 的恐惧而拒绝把相关概念或值聚拢成一个东西会带来 _不同的_ 问题。

所以，与其在"必须 DRY 一切"或"DRY 不好"两个极端任何一边走极端，不如动用脑子，思考你眼前看到的代码。什么在重复？它需要重复吗？如果你把一些重复代码封装到方法里，参数列表看起来还合理吗？它是否感觉不言自明，把"概念"清晰地封装起来了吗？

十次有九次，你看一眼函数的参数列表，如果它看起来凌乱混乱，那就很可能是 DRY 应用得不好。

如果让某段代码 DRY 化感觉很难，你大概是把事情搞得更复杂了；考虑停下来。

DRY 要小心，**但经常练习这件事会改进你的判断**。我鼓励同事们"就试试看"，如果做错了就用版本控制回到安全状态。

<u>**尝试这些事会比讨论它们教你更多**</u>，而版本控制加上良好的自动化测试给了你完美的环境去试验和学习。

### 提取"魔法"值。

> [带有未解释含义的唯一值，或多次出现且（最好）可以替换成命名常量的值](https://en.wikipedia.org/wiki/Magic_number_(programming))

使用提取变量（command+option+v）或常量（command+option+c）给魔法值赋予意义。这可以看作内联重构的逆操作。我经常发现自己在用内联和提取来"切换"代码，以判断哪一种我觉得更易读。

记住，提取重复值也增加了一层 _耦合_。所有使用该值的地方现在都耦合了。考虑下面的代码：

```go
func main() {
	api1Client := http.Client{
		Timeout: 1 * time.Second,
	}
	api2Client := http.Client{
		Timeout: 1 * time.Second,
	}
	api3Client := http.Client{
		Timeout: 1 * time.Second,
	}
	//etc
}
```

我们在为应用设置一些 HTTP 客户端。这里有一些 _魔法值_，我们可以通过提取一个变量并给它一个有意义的名字来 DRY 化 `Timeout`。

![我提取变量的截图](https://i.imgur.com/4sgUG7L.png)

现在代码看起来像这样

```go
func main() {
	timeout := 1 * time.Second
	api1Client := http.Client{
		Timeout: timeout,
	}
	api2Client := http.Client{
		Timeout: timeout,
	}
	api3Client := http.Client{
		Timeout: timeout,
	}
	// etc..
}
```

我们不再有魔法值了；我们给了它一个有意义的名字，但同时也让所有三个客户端 **共享同一个超时**。这 _可能_ 是你想要的；重构是相当依上下文而定的，但这是需要警惕的事情。

如果你能熟练使用 IDE，可以通过 _内联_ 重构让客户端再次拥有各自独立的 `Timeout` 值。

### 让公开的方法/函数易于浏览

你的代码里有过长的公开方法或函数吗？

用提取方法（`command+option+m`）重构把步骤封装到私有方法/函数中。

下面的代码在创建一个 JSON 字符串并把它转成 `io.Reader` 以便我们能通过 HTTP 请求 `POST` 它的过程中，有一些无聊、令人分心的繁文缛节。

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	payload := []byte(`{"name": "` + name + `"}`)

	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer(payload),
	)
	//todo: handle codes, err etc
}
```

首先，使用内联变量重构（command+option+n）把 `payload` 放进 buffer 创建里。

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer([]byte(`{"name": "`+name+`"}`)),
	)
	// etc
}
```

现在，我们可以用提取方法重构（`command+option+m`）把 JSON payload 的创建提取成一个函数，把噪音从方法中移除。

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		createWidgetPayload(name),
	)
	// etc
}
```

公开的方法和函数应该描述它们 *做什么*，而不是 *怎么做*。

> **每当我必须思考才能理解代码在做什么，我就问自己能否重构代码让这种理解更立刻显现**

——Martin Fowler

这有助于你更好地理解整体设计，并让你能就职责提问：

>  这个方法为什么做 X？这难道不应该放在 Y 里吗？

> 这个方法为什么做这么多事？我们能把这整合到别处吗？

私有函数和方法很棒；它们让你把不相关的"怎么做"包装成"做什么"。

#### 但现在我不知道它怎么工作了！

对这种偏好由其他更小函数和方法组成的小函数和方法的重构，一个常见的反对意见是它会让理解代码工作方式变得困难。我对此的直接回答是：

> 你学过用工具有效地浏览代码库吗？

非常刻意地，作为 `CreateWidget` 的 _作者_，我不希望某个特定字符串的创建成为这个方法叙事中的核心角色。99% 的情况下，对读者来说它是分散注意力、无关的噪音。

但是，如果有人 _真的_ 关心，你按 `command+b`（或者你那里的"导航到符号"快捷键）在 `createWidgetPayload` 上……然后读它。再按 `command+left-arrow` 回去。

### 把值的创建移到构造时机。

方法常常需要创建值并使用它们，比如前面 `CreateWidget` 方法中的 `url`。

```go
type WidgetService struct {
	baseURL string
	client  *http.Client
}

func NewWidgetService(baseURL string) *WidgetService {
	client := http.Client{
		Timeout: 10 * time.Second,
	}
	return &WidgetService{baseURL: baseURL, client: &client}
}

func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		createWidgetPayload(name),
	)
	// etc
}
```

你可以应用一种重构技巧：如果一个值的创建 **不依赖于方法的参数**，那么你可以在你的类型里创建一个 _字段_，并在构造函数中计算它。

```go
type WidgetService struct {
	client          *http.Client
	createWidgetURL string
}

func NewWidgetService(baseURL string) *WidgetService {
	client := http.Client{
		Timeout: 10 * time.Second,
	}
	return &WidgetService{
		createWidgetURL: baseURL + "/widgets",
		client:          &client,
	}
}

func (ws *WidgetService) CreateWidget(name string) error {
	req, err := http.NewRequest(
		http.MethodPost,
		ws.createWidgetURL,
		createWidgetPayload(name),
	)
	// etc
}
```

通过把这些移到构造时机，你可以简化你的方法。

#### 比较和对比 `CreateWidget`

最初是

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	payload := []byte(`{"name": "` + name + `"}`)
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer(payload),
	)
	// etc
}

```

通过几次基础的重构，几乎完全用自动化工具驱动，我们得到了

```go
func (ws *WidgetService) CreateWidget(name string) error {
	req, err := http.NewRequest(
		http.MethodPost,
		ws.createWidgetURL,
		createWidgetPayload(name),
	)
	// etc
}
```

这是一个小改进，但毫无疑问读起来更好。如果你练习得多，这种改进几乎不会花你一分钟，并且只要你良好地应用了 TDD，你就有测试的安全网，确保你没破坏任何东西。这些持续的小改进对代码库的长期健康至关重要。

### 尝试移除注释。

> 我们遵循的一条经验法则是：每当我们觉得需要注释什么东西时，就改成写一个方法。

——Martin Fowler

同样，提取方法重构在这里可以是你的朋友。

## 规则的例外

有些代码改进会需要你改测试，但我仍然乐意把它们归入"重构"那一类，即使它打破了规则。

一个简单的例子是用 `shift+F6` 重命名一个公开符号（比如方法、类型或函数）。这当然会同时改动生产代码和测试代码。

不过，由于这是一种 **自动化且安全** 的变更，掉进许多人在其他类型 *设计* 变更中陷入的"破坏测试和生产代码—接连引发一连串问题"螺旋的风险是很小的。

因此，任何你能用 IDE/编辑器安全执行的改动，我仍然乐意称之为重构。

## 用你的工具来帮你练习重构。

- 你应该在每次做这些小改动后都跑你的单元测试。我们投入时间让代码可单元测试，几毫秒的反馈循环是其重要好处之一；用上它！
- 依赖版本控制。你不应该羞于尝试想法。如果满意，就提交；如果不满意，就回退。这应该感觉舒服、容易，而不是大事。
- 你越好地利用单元测试和版本控制，*练习* 重构就越容易。一旦你掌握了这种纪律，**你的设计技能会迅速提升**，因为你有可靠且有效的反馈循环和安全网。
- 在我的职业生涯中，太多次我听到开发者抱怨没时间重构；不幸的是，很明显他们花这么多时间是因为他们做这件事没有纪律——而且他们练习得不够多。
- 虽然敲代码从来不是瓶颈，但你应该能用你使用的任何编辑器/IDE 安全且快速地重构。例如，如果你的工具不能让你按一个键就提取变量，你做这件事就会更少，因为它更费力且风险更高。

## 不要请求许可才去重构

重构应该是你工作中频繁发生的事情，是你随时都在做的事情。它也不应该是一个时间黑洞，尤其是如果你做得勤而少。

如果你不重构，你的内部质量会下降，团队的产能会下降，压力会上升。

Martin Fowler 还有一句精彩的话送给我们。

> 然而，除了你非常接近截止日期的情况之外，你不应该因为没时间就推迟重构。多个项目的经验表明，一次重构会带来生产力的提升。没有足够的时间通常是你需要做一些重构的标志。

## 总结

这不是一个详尽的清单，只是一个开始。读 Martin Fowler 的《Refactoring》（第 2 版）成为高手。

练得多了，重构应该是极其快速且安全的，所以没什么不去做的借口。太多人把重构看作一个由别人来做决定的事情，而不是一项需要学习并使其成为日常工作一部分的技能。

我们应该一直追求让代码处于 *示范级* 的状态。

良好的重构会带来更易理解的代码。理解代码意味着更容易发现更好的设计。在有大型函数、不必要的代码重复、深层嵌套等的系统中找到设计要难得多。**频繁的小重构对更好的设计是必要的**。

