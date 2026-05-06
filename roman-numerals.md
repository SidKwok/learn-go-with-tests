# 罗马数字

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/roman-numerals)**

有些公司在面试过程中会让你做 [罗马数字 Kata](http://codingdojo.org/kata/RomanNumerals/)。本章会展示如何用 TDD 来解决它。

我们要写一个函数，把 [阿拉伯数字](https://en.wikipedia.org/wiki/Arabic_numerals)（数字 0 到 9）转换成罗马数字。

如果你没听说过 [罗马数字](https://en.wikipedia.org/wiki/Roman_numerals)，它们是罗马人写数字的方式。

你通过把符号拼在一起来构造数字，每个符号代表一个数

所以 `I` 是"一"，`III` 是三。

看起来很简单，但有几条有趣的规则。`V` 表示五，但 `IV` 是 4（不是 `IIII`）。

`MCMLXXXIV` 是 1984。这看起来很复杂，从一开始就很难想象怎么写代码把它弄出来。

正如本书反复强调的，软件开发者的一项关键技能是尝试识别 _有用_ 功能的"薄薄的纵向切片"，然后**不断迭代**。TDD 工作流能帮助迭代式开发。

所以与其从 1984 开始，我们从 1 开始。

## 先写测试

```go
func TestRomanNumerals(t *testing.T) {
	got := ConvertToRoman(1)
	want := "I"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

如果你跟到本书这里，希望这对你来说已经显得很无聊、很套路了。这是好事。

## 尝试运行测试

```console
./numeral_test.go:6:9: undefined: ConvertToRoman
```

让编译器引导你

## 写最少量的代码让测试能跑，并查看失败的测试输出

创建函数，但还不要让测试通过，始终要确认测试以你预期的方式失败

```go
func ConvertToRoman(arabic int) string {
	return ""
}
```

它现在应该能跑起来

```console
=== RUN   TestRomanNumerals
--- FAIL: TestRomanNumerals (0.00s)
    numeral_test.go:10: got '', want 'I'
FAIL
```

## 写足够的代码让它通过

```go
func ConvertToRoman(arabic int) string {
	return "I"
}
```

## 重构

目前没什么可重构的。

_我知道_ 直接把结果硬编码出来感觉很奇怪，但用 TDD 我们要尽量在"红色"中待得越短越好。它 _感觉上_ 像我们没做出什么成果，但我们已经定义了 API，并有了一个测试捕捉一条规则；即使"真正的"代码非常笨。

现在用这种不安的感觉去写一个新测试，迫使我们写出稍微不那么笨的代码。

## 先写测试

我们可以用子测试把测试分组分得整齐些

```go
func TestRomanNumerals(t *testing.T) {
	t.Run("1 gets converted to I", func(t *testing.T) {
		got := ConvertToRoman(1)
		want := "I"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("2 gets converted to II", func(t *testing.T) {
		got := ConvertToRoman(2)
		want := "II"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

## 尝试运行测试

```console
=== RUN   TestRomanNumerals/2_gets_converted_to_II
    --- FAIL: TestRomanNumerals/2_gets_converted_to_II (0.00s)
        numeral_test.go:20: got 'I', want 'II'
```

不出意料

## 写足够的代码让它通过

```go
func ConvertToRoman(arabic int) string {
	if arabic == 2 {
		return "II"
	}
	return "I"
}
```

是的，看起来我们还没真正解决问题。所以我们需要写更多测试来推动我们前进。

## 重构

我们的测试里有一些重复。当你测试的内容像"给定输入 X，期望得到 Y"这种形式时，你可能应该用表驱动测试。

```go
func TestRomanNumerals(t *testing.T) {
	cases := []struct {
		Description string
		Arabic      int
		Want        string
	}{
		{"1 gets converted to I", 1, "I"},
		{"2 gets converted to II", 2, "II"},
	}

	for _, test := range cases {
		t.Run(test.Description, func(t *testing.T) {
			got := ConvertToRoman(test.Arabic)
			if got != test.Want {
				t.Errorf("got %q, want %q", got, test.Want)
			}
		})
	}
}
```

我们现在可以轻松添加更多用例，而不用再写一堆测试样板。

我们继续推进，去搞定 3

## 先写测试

把下面这条加到我们的 cases 中

```
{"3 gets converted to III", 3, "III"},
```

## 尝试运行测试

```console
=== RUN   TestRomanNumerals/3_gets_converted_to_III
    --- FAIL: TestRomanNumerals/3_gets_converted_to_III (0.00s)
        numeral_test.go:20: got 'I', want 'III'
```

## 写足够的代码让它通过

```go
func ConvertToRoman(arabic int) string {
	if arabic == 3 {
		return "III"
	}
	if arabic == 2 {
		return "II"
	}
	return "I"
}
```

## 重构

OK，我开始不喜欢这些 if 语句了，仔细看代码你能发现，我们其实是在根据 `arabic` 的大小构建一个由 `I` 组成的字符串。

我们"知道"对于更复杂的数字我们要做某种算术运算和字符串拼接。

带着这些想法尝试一次重构，它 _也许不_ 适合最终方案，但没关系。我们随时可以扔掉代码，靠手头的测试重新开始。

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := 0; i < arabic; i++ {
		result.WriteString("I")
	}

	return result.String()
}
```

你可能记得 [`strings.Builder`](https://golang.org/pkg/strings/#Builder)，我们在
[基准测试](iteration.md#benchmarking) 时讨论过

> Builder 用于通过 Write 方法高效地构建字符串，它能最小化内存拷贝。

通常我不会在没有实际性能问题之前就做这种优化，但代码量并不比"手动"拼接字符串多多少，那就用更快的方案吧。

代码看起来更舒服了，并且 _就我们目前所知_ 它描述了领域。

### 罗马人也讲究 DRY……

事情现在开始变得更复杂了。罗马人有他们的智慧，认为重复字符会让数字难读、难数。所以罗马数字有条规则，同一个字符不能连续出现超过 3 次。

取而代之，你取下一个更高的符号，然后通过把一个符号放到它左边来"减去"。不是所有符号都能做减号；只有 I（1）、X（10）和 C（100）。

例如罗马数字中 `5` 是 `V`。要表示 4，你不能写 `IIII`，而要写 `IV`。

## 先写测试

```
{"4 gets converted to IV (can't repeat more than 3 times)", 4, "IV"},
```

## 尝试运行测试

```console
=== RUN   TestRomanNumerals/4_gets_converted_to_IV_(cant_repeat_more_than_3_times)
    --- FAIL: TestRomanNumerals/4_gets_converted_to_IV_(cant_repeat_more_than_3_times) (0.00s)
        numeral_test.go:24: got 'IIII', want 'IV'
```

## 写足够的代码让它通过

```go
func ConvertToRoman(arabic int) string {

	if arabic == 4 {
		return "IV"
	}

	var result strings.Builder

	for i := 0; i < arabic; i++ {
		result.WriteString("I")
	}

	return result.String()
}
```

## 重构

我"不喜欢"我们打破了字符串构建的模式，我想继续沿用它。

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := arabic; i > 0; i-- {
		if i == 4 {
			result.WriteString("IV")
			break
		}
		result.WriteString("I")
	}

	return result.String()
}
```

为了让 4 适配我现在的思路，我从阿拉伯数字开始倒着数，一边走一边把符号加到字符串里。不确定这种方式从长远看是否能行得通，但试试看！

我们让 5 工作起来

## 先写测试

```
{"5 gets converted to V", 5, "V"},
```

## 尝试运行测试

```console
=== RUN   TestRomanNumerals/5_gets_converted_to_V
    --- FAIL: TestRomanNumerals/5_gets_converted_to_V (0.00s)
        numeral_test.go:25: got 'IIV', want 'V'
```

## 写足够的代码让它通过

照搬我们对 4 的做法

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := arabic; i > 0; i-- {
		if i == 5 {
			result.WriteString("V")
			break
		}
		if i == 4 {
			result.WriteString("IV")
			break
		}
		result.WriteString("I")
	}

	return result.String()
}
```

## 重构

像这样在循环中重复，往往是某个抽象呼之欲出的信号。短路循环可以是提升可读性的有效工具，但它也可能在告诉你别的事。

我们在阿拉伯数字上循环，遇到某些符号就 `break`，但我们 _其实_ 是在以一种笨拙的方式从 `i` 中减去。

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for arabic > 0 {
		switch {
		case arabic > 4:
			result.WriteString("V")
			arabic -= 5
		case arabic > 3:
			result.WriteString("IV")
			arabic -= 4
		default:
			result.WriteString("I")
			arabic--
		}
	}

	return result.String()
}
```

- 根据我从代码中读到的信号——这些信号来自一些非常基础场景的测试——我可以看出，要构造罗马数字，我需要在添加符号的同时从 `arabic` 中减去
- `for` 循环不再依赖 `i`，而是会一直构建字符串，直到我们从 `arabic` 中减去了足够多的符号。

我相当确定这种方式对 6（VI）、7（VII）和 8（VIII）也适用。无论如何，把这些用例加到测试套件中并验证（为了简洁这里就不放代码了，如果你不确定可以去 github 看示例）。

9 和 4 遵循同样的规则：我们应该从下一个数字的表示中减去 `I`。10 在罗马数字中表示为 `X`；所以 9 应该是 `IX`。

## 先写测试

```
{"9 gets converted to IX", 9, "IX"},
```
## 尝试运行测试

```console
=== RUN   TestRomanNumerals/9_gets_converted_to_IX
    --- FAIL: TestRomanNumerals/9_gets_converted_to_IX (0.00s)
        numeral_test.go:29: got 'VIV', want 'IX'
```

## 写足够的代码让它通过

我们应该可以采用之前同样的方式

```
case arabic > 8:
    result.WriteString("IX")
    arabic -= 9
```

## 重构

_感觉_ 代码还在告诉我们某处有个重构，但我还不太能看出来，所以我们继续。

代码我也跳过了，但请把 `10` 的测试用例加进去，它应该是 `X`，让它通过后再继续往下读。

下面是我添加的几个测试，因为我有信心代码到 39 都能工作

```
{"10 gets converted to X", 10, "X"},
{"14 gets converted to XIV", 14, "XIV"},
{"18 gets converted to XVIII", 18, "XVIII"},
{"20 gets converted to XX", 20, "XX"},
{"39 gets converted to XXXIX", 39, "XXXIX"},
```

如果你做过 OO 编程，你应该知道你应该用一点怀疑的眼光看待 `switch` 语句。通常你是在命令式代码里捕捉某个概念或数据，而它其实可以被一个类的结构来捕获。

Go 不是严格的 OO，但这并不意味着我们要彻底无视 OO 提供的教训（虽然有些人可能会这样告诉你）。

我们的 switch 语句在描述罗马数字的某些事实以及行为。

我们可以通过把数据和行为解耦来重构。

```go
type RomanNumeral struct {
	Value  int
	Symbol string
}

var allRomanNumerals = []RomanNumeral{
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}

func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for _, numeral := range allRomanNumerals {
		for arabic >= numeral.Value {
			result.WriteString(numeral.Symbol)
			arabic -= numeral.Value
		}
	}

	return result.String()
}
```

这感觉好多了。我们以数据的形式声明了关于这些数字的一些规则，而不是把它们藏在算法里。我们可以看到，我们只是依次处理阿拉伯数字，尝试在符合时把符号加到结果里。

这种抽象对更大的数字也行得通吗？把测试套件扩展到罗马数字 50（即 `L`）。

下面是一些测试用例，试着让它们通过吧。

```
{"40 gets converted to XL", 40, "XL"},
{"47 gets converted to XLVII", 47, "XLVII"},
{"49 gets converted to XLIX", 49, "XLIX"},
{"50 gets converted to L", 50, "L"},
```

需要帮助？你可以在 [这个 gist](https://gist.github.com/pamelafox/6c7b948213ba55332d86efd0f0b037de) 里看看要加哪些符号。

## 接下来！

下面是剩下的符号

| Arabic | Roman |
| ------ | :---: |
| 100    |   C   |
| 500    |   D   |
| 1000   |   M   |

剩下的符号采用同样的方式，应该只是给测试和符号数组添加数据的事。

你的代码能处理 `1984`：`MCMLXXXIV` 吗？

下面是我最终的测试套件

```go
func TestRomanNumerals(t *testing.T) {
	cases := []struct {
		Arabic int
		Roman  string
	}{
		{Arabic: 1, Roman: "I"},
		{Arabic: 2, Roman: "II"},
		{Arabic: 3, Roman: "III"},
		{Arabic: 4, Roman: "IV"},
		{Arabic: 5, Roman: "V"},
		{Arabic: 6, Roman: "VI"},
		{Arabic: 7, Roman: "VII"},
		{Arabic: 8, Roman: "VIII"},
		{Arabic: 9, Roman: "IX"},
		{Arabic: 10, Roman: "X"},
		{Arabic: 14, Roman: "XIV"},
		{Arabic: 18, Roman: "XVIII"},
		{Arabic: 20, Roman: "XX"},
		{Arabic: 39, Roman: "XXXIX"},
		{Arabic: 40, Roman: "XL"},
		{Arabic: 47, Roman: "XLVII"},
		{Arabic: 49, Roman: "XLIX"},
		{Arabic: 50, Roman: "L"},
		{Arabic: 100, Roman: "C"},
		{Arabic: 90, Roman: "XC"},
		{Arabic: 400, Roman: "CD"},
		{Arabic: 500, Roman: "D"},
		{Arabic: 900, Roman: "CM"},
		{Arabic: 1000, Roman: "M"},
		{Arabic: 1984, Roman: "MCMLXXXIV"},
		{Arabic: 3999, Roman: "MMMCMXCIX"},
		{Arabic: 2014, Roman: "MMXIV"},
		{Arabic: 1006, Roman: "MVI"},
		{Arabic: 798, Roman: "DCCXCVIII"},
	}
	for _, test := range cases {
		t.Run(fmt.Sprintf("%d gets converted to %q", test.Arabic, test.Roman), func(t *testing.T) {
			got := ConvertToRoman(test.Arabic)
			if got != test.Roman {
				t.Errorf("got %q, want %q", got, test.Roman)
			}
		})
	}
}
```

- 我去掉了 `description`，因为 _数据本身_ 描述的信息足够了。
- 我加了一些其他的边界用例来给我多一点信心。表驱动测试做这种事成本非常低。

我没改动算法，只更新了 `allRomanNumerals` 数组。

```go
var allRomanNumerals = []RomanNumeral{
	{1000, "M"},
	{900, "CM"},
	{500, "D"},
	{400, "CD"},
	{100, "C"},
	{90, "XC"},
	{50, "L"},
	{40, "XL"},
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}
```

## 解析罗马数字

我们还没完。接下来我们要写一个函数，把罗马数字 _转换_ 回 `int`

## 先写测试

我们可以稍作重构来复用测试用例

把 `cases` 变量从测试函数中移出，放到一个 `var` 块里作为包级变量。

```go
func TestConvertingToArabic(t *testing.T) {
	for _, test := range cases[:1] {
		t.Run(fmt.Sprintf("%q gets converted to %d", test.Roman, test.Arabic), func(t *testing.T) {
			got := ConvertToArabic(test.Roman)
			if got != test.Arabic {
				t.Errorf("got %d, want %d", got, test.Arabic)
			}
		})
	}
}
```

注意我用切片功能现在只跑其中一个测试（`cases[:1]`），因为试图一次让所有这些测试都通过跨度太大

## 尝试运行测试

```console
./numeral_test.go:60:11: undefined: ConvertToArabic
```

## 写最少量的代码让测试能跑，并查看失败的测试输出

加上新函数定义

```go
func ConvertToArabic(roman string) int {
	return 0
}
```

测试现在应该能跑并失败

```console
--- FAIL: TestConvertingToArabic (0.00s)
    --- FAIL: TestConvertingToArabic/'I'_gets_converted_to_1 (0.00s)
        numeral_test.go:62: got 0, want 1
```

## 写足够的代码让它通过

你知道该怎么做

```go
func ConvertToArabic(roman string) int {
	return 1
}
```

接下来，把测试中的切片索引改到下一个测试用例（如 `cases[:2]`）。用你能想到的最笨的代码让它通过，然后第三个用例也继续写笨代码（最棒的书莫过于此，对吧？）。下面是我的笨代码。

```go
func ConvertToArabic(roman string) int {
	if roman == "III" {
		return 3
	}
	if roman == "II" {
		return 2
	}
	return 1
}
```

通过这些 _能跑的真代码_ 的笨拙，我们可以像之前一样开始看出一种模式。我们需要遍历输入并构建 _某种东西_，这次是一个总和。

```go
func ConvertToArabic(roman string) int {
	total := 0
	for range roman {
		total++
	}
	return total
}
```

## 先写测试

接下来我们移到 `cases[:4]`（`IV`），它现在会失败因为它返回 2，那是字符串的长度。

## 写足够的代码让它通过

```go
// earlier..
var allRomanNumerals = RomanNumerals{
	{1000, "M"},
	{900, "CM"},
	{500, "D"},
	{400, "CD"},
	{100, "C"},
	{90, "XC"},
	{50, "L"},
	{40, "XL"},
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}

// later..
func ConvertToArabic(roman string) int {
	var arabic = 0

	for _, numeral := range allRomanNumerals {
		for strings.HasPrefix(roman, numeral.Symbol) {
			arabic += numeral.Value
			roman = strings.TrimPrefix(roman, numeral.Symbol)
		}
	}

	return arabic
}
```

它基本上就是 `ConvertToRoman(int)` 算法的反向实现。这里我们在给定的罗马数字字符串上循环：
- 我们从字符串的开头查找罗马数字符号，按从大到小的顺序、来自 `allRomanNumerals`。
- 如果找到前缀，就把它的值加到 `arabic`，然后把这个前缀从字符串里删掉。

最后我们返回总和作为阿拉伯数字。

`HasPrefix(s, prefix)` 检查字符串 `s` 是否以 `prefix` 开头，`TrimPrefix(s, prefix)` 把 `prefix` 从 `s` 中去掉，这样我们就能继续处理剩下的罗马数字符号。它对 `IV` 和所有其他测试用例都有效。

你可以把它实现成递归函数，那样会更优雅（在我看来），但可能慢一些。这就留给你和一些 `Benchmark...` 测试去探索吧。

既然我们有了把阿拉伯数字转成罗马数字以及反过来的函数，我们可以把测试再往前推一步：

## 基于属性的测试介绍

本章我们一直在处理罗马数字领域的几条规则

- 不能有超过 3 个连续相同的符号
- 只有 I（1）、X（10）和 C（100）可以做"减号"
- 取 `ConvertToRoman(N)` 的结果传给 `ConvertToArabic`，应当返回 `N`

到目前为止我们写的测试可以叫"基于示例"的测试，我们提供 _示例_ 让工具去验证。

如果我们能把这些关于领域的规则拿出来，让它们对我们的代码进行某种检验，会怎样？

基于属性的测试帮我们做到这一点：它向你的代码扔随机数据，并验证你描述的规则始终成立。很多人以为基于属性的测试主要是关于随机数据，但他们想错了。基于属性的测试真正的挑战是 _深入_ 理解你的领域，这样才能写出这些属性。

废话不多说，看代码

> **⚠️ Linux 用户：**请**不要**立刻运行下面的测试。它很可能让你的系统冻死（需要硬重启）。
>
> <details>
> <summary>点击这里查看原因（技术解释）</summary>
>
> `testing/quick` 包会生成最大到 `int64` 上限的随机整数。我们当前朴素的实现会试图在内存里构建一个那么长的字符串（千万亿个字符）。
>
> 虽然 macOS 和 Windows 通常处理得还算优雅（UI 仍然响应），但 Linux 内核通常会遇到"swap thrashing"，导致整个系统在进程被终结之前就冻住。
> </details>

```go
func TestPropertiesOfConversion(t *testing.T) {
	assertion := func(arabic int) bool {
		roman := ConvertToRoman(arabic)
		fromRoman := ConvertToArabic(roman)
		return fromRoman == arabic
	}

	if err := quick.Check(assertion, nil); err != nil {
		t.Error("failed checks", err)
	}
}
```

### 这条属性的依据

我们的第一个测试会检查：如果我们把一个数转成罗马数字，再用另一个函数把它转回来，我们应该得到原来的数字。

- 给定一个随机数（比如 `4`）。
- 用这个随机数调用 `ConvertToRoman`（如果是 `4` 应该返回 `IV`）。
- 把上面的结果传给 `ConvertToArabic`。
- 上面应该给我们原来的输入（`4`）。

这感觉是一个能给我们建立信心的好测试，因为只要其中一个有 bug，它就会挂掉。它能通过的唯一方式是两个函数有相同种类的 bug；这并非不可能，但感觉不太可能。

### 技术解释

 我们使用了标准库里的 [testing/quick](https://golang.org/pkg/testing/quick/) 包

 从下往上读，我们给 `quick.Check` 一个函数，它会用一系列随机输入运行该函数，如果函数返回 `false` 就视为检查失败。

 我们上面的 `assertion` 函数接受一个随机数，运行我们的函数来检验这条属性。

### 跑一下我们的测试

 试着运行；你的电脑可能会卡一会儿，所以等不下去就把它干掉吧 :)

 怎么回事？把下面的代码加到 assertion 中。

 ```go
assertion := func(arabic int) bool {
	if arabic < 0 || arabic > 3999 {
		log.Println(arabic)
		return true
	}
	roman := ConvertToRoman(arabic)
	fromRoman := ConvertToArabic(roman)
	return fromRoman == arabic
}
```

你应该会看到类似这样的输出：

```console
=== RUN   TestPropertiesOfConversion
2019/07/09 14:41:27 6849766357708982977
2019/07/09 14:41:27 -7028152357875163913
2019/07/09 14:41:27 -6752532134903680693
2019/07/09 14:41:27 4051793897228170080
2019/07/09 14:41:27 -1111868396280600429
2019/07/09 14:41:27 8851967058300421387
2019/07/09 14:41:27 562755830018219185
```

仅仅运行这个非常简单的属性，就暴露了我们实现中的一个缺陷。我们用 `int` 作为输入，但是：
- 罗马数字不能表示负数
- 由于最多 3 个连续符号的规则，我们无法表示大于 3999 的值（[嗯，差不多吧](https://www.quora.com/Which-is-the-maximum-number-in-Roman-numerals)），而 `int` 的最大值远大于 3999。

这太棒了！我们被迫更深入地思考自己的领域，这正是基于属性的测试的真正威力。

显然 `int` 不是一个好的类型。如果我们试一种更合适的呢？

### [`uint16`](https://golang.org/pkg/builtin/#uint16)

Go 有 _无符号整数_ 的类型，意味着它们不能为负；这立刻排除了我们代码里的一类 bug。再加 16，意味着它是 16 位整数，最大可以存到 `65535`，仍然太大，但已经更接近我们需要的了。

试着把代码改成用 `uint16` 而不是 `int`。我也更新了测试中的 `assertion` 让它更直观。

```go
assertion := func(arabic uint16) bool {
	if arabic > 3999 {
		return true
	}
	t.Log("testing", arabic)
	roman := ConvertToRoman(arabic)
	fromRoman := ConvertToArabic(roman)
	return fromRoman == arabic
}
```
注意我们现在用 testing 框架的 `log` 方法记录输入。请确保运行 `go test` 时带上 `-v` 标志（`go test -v`），这样才能看到额外输出。

如果你跑测试，它们现在能真正运行了，你可以看到正在测试的内容。你可以多跑几次，看看我们的代码在各种值下能很好地坚持住！这给了我很大的信心，相信我们的代码按预期工作。

`quick.Check` 默认运行次数是 100，但你可以通过配置改变。

```go
if err := quick.Check(assertion, &quick.Config{
	MaxCount: 1000,
}); err != nil {
	t.Error("failed checks", err)
}
```

### 进一步的工作

- 你能写出基于属性的测试来检查我们描述的其他属性吗？
- 你能想出办法让别人无法用大于 3999 的数字调用我们的代码吗？
    - 你可以返回一个错误
    - 或者创建一个不能表示 > 3999 的新类型
        - 你认为哪种最好？

## 总结

### 用迭代式开发更多 TDD 练习

把 1984 转换成 MCMLXXXIV 的代码一开始让你觉得有点吓人吗？我也是，而我已经写软件相当长时间了。

诀窍一如既往：**从简单的事情开始**，迈**小步**。

在这个过程中，我们没在任何时候做大跳跃、做大规模重构，或陷入混乱。

我能听到有人愤世嫉俗地说"这只是一个 kata 练习啊"。我无法反驳，但我对每个我做的项目都是这种方式。我从不会在第一步就发布一个庞大的分布式系统，我会找团队能交付的最简单的东西（通常是一个 "Hello world" 网站），然后以可管理的小块迭代功能，就像我们这里做的一样。

技能在于知道 _如何_ 切分工作，这需要练习，也需要可爱的 TDD 帮你前行。

### 基于属性的测试

- 内置在标准库中
- 如果你能用代码描述你的领域规则，它们是给你建立更强信心的极佳工具
- 强迫你深入思考你的领域
- 可能很好地补充你的测试套件

## 后记

本书依赖于社区宝贵的反馈。
[Dave](http://github.com/gypsydave5) 在几乎每一章节都给予了巨大的帮助。
但他对我在这一章节中使用'阿拉伯数字'非常不爽，所以本着完全公开透明的精神，下面是他的话。

> 我打算写一下为什么 `int` 类型的值并不真的是"阿拉伯数字"。这可能有点过分较真，
> 所以如果你让我滚开，我完全理解。
>
> _digit_（数字符号）是用于表示数字的字符——这个词来自拉丁语的"手指"，因为我们通常有十根。
> 在阿拉伯（也叫印度-阿拉伯）数字系统中，有十个数字符号。这些阿拉伯数字符号是：
>
> ```console
>   0 1 2 3 4 5 6 7 8 9
> ```
>
> _numeral_（数字）是用一组数字符号来表示一个数。
> 阿拉伯数字（Arabic numeral）是用阿拉伯数字符号在十进制位置数字系统中表示的数。
> 我们说"位置"是因为每个数字符号根据它在数字中的位置具有不同的值。所以
>
> ```console
>   1337
> ```
>
> 这里的 `1` 因为是四位数中的第一位，所以表示一千。
>
> 罗马数字使用更少的数字符号（`I`、`V` 等等）作为构造数字的值。
> 它有一点位置性的东西，但大部分情况下 `I` 永远表示"一"。
>
> 那么，在此基础上，`int` 是一个"阿拉伯数"吗？数的概念与它的表示完全没有联系——
> 我们可以这样问自己：下面这个数的正确表示是什么？
>
> ```console
> 255
> 11111111
> two-hundred and fifty-five
> FF
> 377
> ```
>
> 是的，这是个圈套问题。它们都是对的。它们分别是十进制、二进制、英文、十六进制和八进制
> 数字系统中同一个数的表示。
>
> 数作为数字的表示与它作为数的属性是 _独立的_，我们看 Go 的整数字面量就能看出来：
>
> ```go
> 	0xFF == 255 // true
> ```
>
> 以及我们如何用格式串打印整数：
>
> ```go
> n := 255
> fmt.Printf("%b %c %d %o %q %x %X %U", n, n, n, n, n, n, n, n)
> // 11111111 ÿ 255 377 'ÿ' ff FF U+00FF
> ```
>
> 同一个整数我们既可以写成十六进制，也可以写成阿拉伯（十进制）数字。
>
> 所以当函数签名长这样 `ConvertToRoman(arabic int) string` 的时候，
> 它在对它如何被调用做出一些假设。因为有时 `arabic` 会被写成十进制整数字面量
>
> ```go
> 	ConvertToRoman(255)
> ```
>
> 但它也可能被写成
>
> ```go
> 	ConvertToRoman(0xFF)
> ```
>
> 实际上，我们根本不是在从阿拉伯数字"转换"，而是在"打印"——
> 把一个 `int` 表示为罗马数字——而 `int` 不是数字（不管是阿拉伯还是其他），
> 它们就是数。`ConvertToRoman` 函数更像 `strconv.Itoa`，它把 `int` 变成 `string`。
>
> 但 kata 的所有其他版本都不在意这种区分，所以
> :shrug:
