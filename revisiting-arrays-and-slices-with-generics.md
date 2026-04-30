# 用泛型重看数组和切片

**[本章的代码是从「数组和切片」那一章延续过来的，可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/arrays)**

看一下我们在[数组和切片](arrays-and-slices.md)中写的 `SumAll` 和 `SumAllTails`。如果你没有自己的版本，请从[数组和切片](arrays-and-slices.md)章节连同测试一起把代码复制过来。

```go
// Sum calculates the total from a slice of numbers.
func Sum(numbers []int) int {
	var sum int
	for _, number := range numbers {
		sum += number
	}
	return sum
}

// SumAllTails calculates the sums of all but the first number given a collection of slices.
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		if len(numbers) == 0 {
			sums = append(sums, 0)
		} else {
			tail := numbers[1:]
			sums = append(sums, Sum(tail))
		}
	}

	return sums
}
```

你看到一个反复出现的模式了吗？

- 创建某种"初始"结果值。
- 遍历集合，对结果和切片中的下一项应用某种操作（或函数），为结果设置一个新值
- 返回结果。

这个想法在函数式编程圈里经常被讨论，常被称为 'reduce' 或 [fold](https://en.wikipedia.org/wiki/Fold_(higher-order_function))。

> 在函数式编程中，fold（也叫 reduce、accumulate、aggregate、compress 或 inject）指的是一类高阶函数，它们分析递归数据结构，并通过给定的组合操作，重新合并对其各个组成部分的递归处理结果，构建出一个返回值。通常 fold 会接受一个组合函数、数据结构的顶层节点，可能还有一些在某些条件下使用的默认值。然后 fold 系统地使用该函数，把数据结构层级中的元素组合起来。

Go 一直有高阶函数，在 1.18 版本开始它也有了[泛型](./generics.md)，所以现在可以定义一些在我们更广泛领域中讨论的这些函数。把头埋在沙子里是没有意义的，这是 Go 生态之外非常常见的抽象，理解它对你有好处。

现在，我知道你们中有些人可能已经在皱眉了。

> Go 应该是简单的

**不要把容易和简单混为一谈**。写循环和复制粘贴代码很容易，但不一定简单。关于"简单"和"容易"的更多内容，请看 [Rich Hickey 的杰作演讲——Simple Made Easy](https://www.youtube.com/watch?v=SxdOUGdseq4)。

**不要把不熟悉和复杂混为一谈**。Fold/reduce 一开始可能听起来吓人、计算机科学味十足，但它实际上只是对一个非常常见操作的抽象。把一个集合合并成一个东西。退一步看，你会意识到你大概 _经常_ 这样做。

## 一次泛型重构

人们常犯的一个错误是，在没有具体使用场景的情况下就开始使用闪亮的新语言特性。他们依靠猜想和瞎猜来指导自己的工作。

幸好我们已经写了"有用"的函数，并围绕它们写了测试，所以我们可以在 TDD 的重构阶段自由地试验想法，并知道无论我们尝试什么，都有单元测试来验证其价值。

把泛型作为重构步骤中简化代码的工具来使用，比起预先抽象，更可能引导你做出有用的改进。

我们可以放心地试，重新跑测试，喜欢这个改动就提交。如果不喜欢，就回退改动。这种试验的自由是 TDD 真正巨大的价值之一。

你应该已经熟悉[上一章](generics.md)的泛型语法了，试着自己写一个 `Reduce` 函数，并在 `Sum` 和 `SumAllTails` 里使用它。

### 提示

如果你先想想函数的参数，会得到一组很小的有效解
  - 你想 reduce 的数组
  - 某种组合函数

"Reduce" 是一个有非常详尽文档的模式，没必要重新发明轮子。[读一下 wiki，特别是列表那一节](https://en.wikipedia.org/wiki/Fold_(higher-order_function))，它应该会提示你需要的另一个参数。

> 实际上，有一个初始值是方便而自然的

### 我第一版的 `Reduce`

```go
func Reduce[A any](collection []A, f func(A, A) A, initialValue A) A {
	var result = initialValue
	for _, x := range collection {
		result = f(result, x)
	}
	return result
}
```

Reduce 抓住了这个模式的 _本质_，它是一个接受一个集合、一个累积函数、一个初始值并返回单个值的函数。没有围绕具体类型的杂乱干扰。

如果你理解泛型语法，你应该能毫无问题地理解这个函数做什么。通过使用公认的术语 `Reduce`，来自其他语言的程序员也能理解这个意图。

### 用法

```go
// Sum calculates the total from a slice of numbers.
func Sum(numbers []int) int {
	add := func(acc, x int) int { return acc + x }
	return Reduce(numbers, add, 0)
}

// SumAllTails calculates the sums of all but the first number given a collection of slices.
func SumAllTails(numbers ...[]int) []int {
	sumTail := func(acc, x []int) []int {
		if len(x) == 0 {
			return append(acc, 0)
		} else {
			tail := x[1:]
			return append(acc, Sum(tail))
		}
	}

	return Reduce(numbers, sumTail, []int{})
}
```

`Sum` 和 `SumAllTails` 现在在它们各自第一行声明的函数中描述了自身计算的行为。在集合上运行计算这件事被抽象到 `Reduce` 中。

## reduce 的更多应用

通过测试，我们可以试用我们的 reduce 函数，看看它有多可复用。我从上一章把我们的泛型断言函数复制过来了。

```go
func TestReduce(t *testing.T) {
	t.Run("multiplication of all elements", func(t *testing.T) {
		multiply := func(x, y int) int {
			return x * y
		}

		AssertEqual(t, Reduce([]int{1, 2, 3}, multiply, 1), 6)
	})

	t.Run("concatenate strings", func(t *testing.T) {
		concatenate := func(x, y string) string {
			return x + y
		}

		AssertEqual(t, Reduce([]string{"a", "b", "c"}, concatenate, ""), "abc")
	})
}
```

### 零值

在乘法的例子里，我们展示了为什么要把默认值作为参数传给 `Reduce`。如果我们依赖 Go 对 `int` 的默认值 0，我们就会把初始值乘以 0，然后再乘以后续的值，这样你只能得到 0。把它设为 1，切片中的第一个元素会保持不变，后续会乘以下一个元素。

如果你想在你的极客朋友面前显得聪明，你可以把这个叫做[单位元](https://en.wikipedia.org/wiki/Identity_element)。

> 在数学中，作用于某个集合上的二元运算的单位元（或称中性元）是该集合中的一个元素，当对该集合中的任何元素施加该运算时，单位元保持其不变。

加法中，单位元是 0。

`1 + 0 = 1`

乘法中，是 1。

`1 * 1 = 1`

## 如果我们希望 reduce 成与 `A` 不同的类型怎么办？

假设我们有一个交易列表 `Transaction`，我们想要一个函数，接受这些交易加上一个名字，来算出他们的银行余额。

让我们遵循 TDD 流程。

## 先写测试

```go
func TestBadBank(t *testing.T) {
	transactions := []Transaction{
		{
			From: "Chris",
			To:   "Riya",
			Sum:  100,
		},
		{
			From: "Adil",
			To:   "Chris",
			Sum:  25,
		},
	}

	AssertEqual(t, BalanceFor(transactions, "Riya"), 100)
	AssertEqual(t, BalanceFor(transactions, "Chris"), -75)
	AssertEqual(t, BalanceFor(transactions, "Adil"), -25)
}
```

## 试着运行测试
```
# github.com/quii/learn-go-with-tests/arrays/v8 [github.com/quii/learn-go-with-tests/arrays/v8.test]
./bad_bank_test.go:6:20: undefined: Transaction
./bad_bank_test.go:18:14: undefined: BalanceFor
```

## 写最少的代码让测试能运行，并检查失败的测试输出

我们还没有类型或函数，把它们加上让测试跑起来。

```go
type Transaction struct {
	From string
	To   string
	Sum  float64
}

func BalanceFor(transactions []Transaction, name string) float64 {
	return 0.0
}
```

当你运行测试，应该看到下面这样：

```
=== RUN   TestBadBank
    bad_bank_test.go:19: got 0, want 100
    bad_bank_test.go:20: got 0, want -75
    bad_bank_test.go:21: got 0, want -25
--- FAIL: TestBadBank (0.00s)
```

## 写足够的代码让它通过

我们先假装没有 `Reduce` 函数来写代码。

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	var balance float64
	for _, t := range transactions {
		if t.From == name {
			balance -= t.Sum
		}
		if t.To == name {
			balance += t.Sum
		}
	}
	return balance
}
```

## 重构

到这里，养成一些版本控制的纪律，提交你的工作。我们有了能工作的软件，准备挑战 Monzo、Barclays 等等。

我们的工作提交后，就可以自由地玩转它，在重构阶段试验不同的想法。说实话，我们现在的代码并不算糟，但为了演示，我想用 `Reduce` 展示同样的代码。

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	adjustBalance := func(currentBalance float64, t Transaction) float64 {
		if t.From == name {
			return currentBalance - t.Sum
		}
		if t.To == name {
			return currentBalance + t.Sum
		}
		return currentBalance
	}
	return Reduce(transactions, adjustBalance, 0.0)
}
```

但这编译不过。

```
./bad_bank.go:19:35: type func(acc float64, t Transaction) float64 of adjustBalance does not match inferred type func(Transaction, Transaction) Transaction for func(A, A) A
```

原因是我们试图把 reduce 的结果变成与集合类型 _不同_ 的类型。这听起来吓人，但实际上只需要我们调整 `Reduce` 的类型签名就能让它工作。我们不必改函数体，也不必改任何已有的调用方。

```go
func Reduce[A, B any](collection []A, f func(B, A) B, initialValue B) B {
	var result = initialValue
	for _, x := range collection {
		result = f(result, x)
	}
	return result
}
```

我们加了第二个类型约束，这让我们放宽了 `Reduce` 的约束。这允许人们把 `A` 的集合 `Reduce` 成 `B`。在我们这个例子中，是从 `Transaction` 到 `float64`。

这让 `Reduce` 更通用、更可复用，并且仍然类型安全。如果你再次运行测试，它们应该能编译并通过。

## 扩展银行

为了好玩，我想改进银行代码的人体工学。为了简洁，我省略了 TDD 过程。

```go
func TestBadBank(t *testing.T) {
	var (
		riya  = Account{Name: "Riya", Balance: 100}
		chris = Account{Name: "Chris", Balance: 75}
		adil  = Account{Name: "Adil", Balance: 200}

		transactions = []Transaction{
			NewTransaction(chris, riya, 100),
			NewTransaction(adil, chris, 25),
		}
	)

	newBalanceFor := func(account Account) float64 {
		return NewBalanceFor(account, transactions).Balance
	}

	AssertEqual(t, newBalanceFor(riya), 200)
	AssertEqual(t, newBalanceFor(chris), 0)
	AssertEqual(t, newBalanceFor(adil), 175)
}
```

下面是更新后的代码

```go
package main

type Transaction struct {
	From string
	To   string
	Sum  float64
}

func NewTransaction(from, to Account, sum float64) Transaction {
	return Transaction{From: from.Name, To: to.Name, Sum: sum}
}

type Account struct {
	Name    string
	Balance float64
}

func NewBalanceFor(account Account, transactions []Transaction) Account {
	return Reduce(
		transactions,
		applyTransaction,
		account,
	)
}

func applyTransaction(a Account, transaction Transaction) Account {
	if transaction.From == a.Name {
		a.Balance -= transaction.Sum
	}
	if transaction.To == a.Name {
		a.Balance += transaction.Sum
	}
	return a
}
```

我感觉这真的展示了使用 `Reduce` 这类概念的力量。`NewBalanceFor` 感觉更具 _声明性_，描述了 _发生了什么_，而不是 _怎么发生_。我们读代码时常常在很多文件之间穿梭，想要理解的是 _发生了什么_，而不是 _怎么发生_，这种代码风格能很好地满足这一点。

如果我想深入细节，我可以看，并且我可以看到 `applyTransaction` 的 _业务逻辑_，而不必担心循环和可变状态；`Reduce` 单独处理了这些。


### Fold/reduce 是相当通用的

`Reduce`（或 `Fold`）的可能性是无限的™️。它是一个常见的模式是有原因的，它不仅仅用于算术或字符串拼接。试一些其他的应用。

- 何不把一些 `color.RGBA` 混合成单一的颜色？
- 统计一次投票中的票数，或购物篮中的物品数。
- 几乎任何涉及处理列表的事情。

## Find

既然 Go 有了泛型，把它和高阶函数结合起来，我们就可以减少项目中大量样板代码，让我们的系统更易于理解和管理。

不再需要为你想搜索的每种集合类型写专门的 `Find` 函数，而是复用或编写一个 `Find` 函数。如果你理解了上面的 `Reduce` 函数，写一个 `Find` 函数就是小菜一碟。

下面是一个测试

```go
func TestFind(t *testing.T) {
	t.Run("find first even number", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

		firstEvenNumber, found := Find(numbers, func(x int) bool {
			return x%2 == 0
		})
		AssertTrue(t, found)
		AssertEqual(t, firstEvenNumber, 2)
	})
}
```

下面是实现

```go
func Find[A any](items []A, predicate func(A) bool) (value A, found bool) {
	for _, v := range items {
		if predicate(v) {
			return v, true
		}
	}
	return
}
```

同样，因为它接受一个泛型类型，我们可以以多种方式复用它

```go
type Person struct {
	Name string
}

t.Run("Find the best programmer", func(t *testing.T) {
	people := []Person{
		Person{Name: "Kent Beck"},
		Person{Name: "Martin Fowler"},
		Person{Name: "Chris James"},
	}

	king, found := Find(people, func(p Person) bool {
		return strings.Contains(p.Name, "Chris")
	})

	AssertTrue(t, found)
	AssertEqual(t, king, Person{Name: "Chris James"})
})
```

如你所见，这段代码无懈可击。

## 总结

把握得当，这类高阶函数会让你的代码更易读、更易维护，但记住经验法则：

用 TDD 流程驱动出你实际需要的、真实的、特定的行为，然后在重构阶段你 _可能_ 会发现一些有用的抽象来帮你整理代码。

练习把 TDD 和良好的版本控制习惯结合起来。当测试通过时提交你的工作，_然后_ 再尝试重构。这样如果你搞乱了，你能轻易回到能工作的状态。

### 命名很重要

在 Go 之外做点研究，这样你就不会重新发明已经有了名字的模式。

写一个函数把 `A` 的集合转换成 `B`？别叫它 `Convert`，这是 [`Map`](https://en.wikipedia.org/wiki/Map_(higher-order_function))。对这些东西使用"恰当"的名字会减少他人的认知负担，并让你更容易在搜索引擎上学到更多。

### 这感觉不像 Go 的惯用法？

试着保持开放的心态。

虽然 Go 的惯用法不会、也不应该因为泛型的发布而 _彻底_ 改变，但惯用法 _会_ 改变——因为语言改变了！这不应该是有争议的观点。

只说

> 这不是惯用法

而没有任何更多细节，不是一个可执行的、有用的说法。尤其是在讨论新语言特性时。

和你的同事根据代码本身的优劣讨论模式和风格，而不是凭教条。只要你有设计良好的测试，你总是可以重构、调整，逐步明白什么对你和你的团队最有效。

### 资源

Fold 是计算机科学中真正的基础。如果你想深入了解，下面是一些有趣的资源
- [维基百科：Fold](https://en.wikipedia.org/wiki/Fold)
- [关于 fold 的普适性和表达力的教程](http://www.cs.nott.ac.uk/~pszgmh/fold.pdf)
