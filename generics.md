# 泛型

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/generics)**

本章将带你入门泛型，打消你对它的疑虑，并让你了解将来如何用它来简化你的代码。读完之后，你会知道如何写：

- 一个接受泛型参数的函数
- 一个泛型数据结构


## 我们自己的测试辅助函数（`AssertEqual`、`AssertNotEqual`）

为了探索泛型，我们将编写一些测试辅助函数。

### 对整数做断言

让我们从一些基础的东西开始，然后一步步迭代到目标

```go
import "testing"

func TestAssertFunctions(t *testing.T) {
	t.Run("asserting on integers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})
}

func AssertEqual(t *testing.T, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d, want %d", got, want)
	}
}

func AssertNotEqual(t *testing.T, got, want int) {
	t.Helper()
	if got == want {
		t.Errorf("didn't want %d", got)
	}
}
```


### 对字符串做断言

能对整数的相等性做断言很棒，但如果我们想对 `string` 做断言呢？

```go
t.Run("asserting on strings", func(t *testing.T) {
	AssertEqual(t, "hello", "hello")
	AssertNotEqual(t, "hello", "Grace")
})
```

你会得到一个错误

```
# github.com/quii/learn-go-with-tests/generics [github.com/quii/learn-go-with-tests/generics.test]
./generics_test.go:12:18: cannot use "hello" (untyped string constant) as int value in argument to AssertEqual
./generics_test.go:13:21: cannot use "hello" (untyped string constant) as int value in argument to AssertNotEqual
./generics_test.go:13:30: cannot use "Grace" (untyped string constant) as int value in argument to AssertNotEqual
```

如果你认真读这个错误，会发现编译器在抱怨我们试图把一个 `string` 传给一个期望 `integer` 的函数。

#### 类型安全回顾

如果你读过本书前面的章节，或者有静态类型语言的经验，这应该不会让你惊讶。Go 编译器希望你在写函数、结构体等时，描述清楚你想处理的类型。

你不能把一个 `string` 传给一个期望 `integer` 的函数。

虽然这感觉有点繁琐，但它能带来巨大的帮助。通过描述这些约束，你：

- 让函数实现更简单。通过告诉编译器你处理的是什么类型，你**约束了可能的有效实现的数量**。你不能"加"一个 `Person` 和一个 `BankAccount`。你不能把一个 `integer` 大写化。在软件中，约束往往非常有帮助。
- 防止你不小心把数据传给一个你本不打算传的函数。

Go 提供了一种方式让你可以更抽象地处理类型，那就是[接口](./structs-methods-and-interfaces.md)，这样你就可以设计一些不接受具体类型，而是接受提供你需要行为的类型的函数。这给了你一些灵活性，同时保留了类型安全。

### 一个能接受字符串或整数的函数？（甚至别的东西）

Go 让函数更灵活的另一个选项是把参数的类型声明为 `interface{}`，意思是"任何东西"。

试着把签名改成使用这个类型。

```go
func AssertEqual(got, want interface{})

func AssertNotEqual(got, want interface{})

```

测试现在应该能编译并通过了。如果你试着让它们失败，你会看到输出有点糟糕，因为我们用整数的 `%d` 格式化串来打印消息，把它们改成通用的 `%+v` 格式，能让任何类型的值都有更好的输出。

### `interface{}` 的问题

我们的 `AssertX` 函数相当朴素，但概念上和其他[流行的库提供这一功能的方式](https://github.com/matryer/is/blob/master/is.go#L150)没什么不同

```go
func (is *I) Equal(a, b interface{})
```

那问题在哪？

通过使用 `interface{}`，编译器在我们写代码时帮不上忙，因为我们没有告诉它任何关于传给函数的东西的类型的有用信息。试试比较两个不同的类型。

```go
AssertEqual(1, "1")
```

在这种情况下，我们侥幸过关；测试能编译，并且如我们所愿地失败了，尽管错误消息 `got 1, want 1` 不太清楚；但我们真的想能用字符串和整数比较吗？比较一个 `Person` 和一个 `Airport` 又如何？

写接受 `interface{}` 的函数会非常具有挑战性，且容易出 bug，因为我们 _失去了_ 约束，并且在编译期没有任何关于我们要处理什么类型数据的信息。

这意味着 **编译器帮不了我们**，相反我们更可能遇到 **运行时错误**，这些错误可能影响用户、导致服务中断，或更糟。

通常开发者必须使用反射来实现这些 *呃哼* 通用函数，这会让代码读写都很复杂，并且会损害你程序的性能。

## 用泛型实现我们自己的测试辅助函数

理想情况下，我们不希望为我们处理的每种类型都做一个特定的 `AssertX` 函数。我们希望能有 _一个_ `AssertEqual` 函数，它能处理 _任何_ 类型，但不允许你拿[苹果和橘子](https://en.wikipedia.org/wiki/Apples_and_oranges)做比较。

泛型为我们提供了一种制作抽象（像接口一样）的方法，它让我们 **描述我们的约束**。它允许我们写出灵活程度类似 `interface{}` 的函数，但保留类型安全，并为调用者提供更好的开发体验。

```go
func TestAssertFunctions(t *testing.T) {
	t.Run("asserting on integers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})

	t.Run("asserting on strings", func(t *testing.T) {
		AssertEqual(t, "hello", "hello")
		AssertNotEqual(t, "hello", "Grace")
	})

	// AssertEqual(t, 1, "1") // 取消注释来看错误
}

func AssertEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
}

func AssertNotEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got == want {
		t.Errorf("didn't want %v", got)
	}
}
```

要在 Go 中写泛型函数，你需要提供"类型参数"，这只是一种花哨的说法，意思是"描述你的泛型类型并给它一个标签"。

在我们的例子中，类型参数的类型是 `comparable`，我们给它的标签是 `T`。这个标签让我们可以描述函数参数的类型（`got, want T`）。

我们用 `comparable`，是因为我们想告诉编译器：在我们的函数里，我们想对类型为 `T` 的东西使用 `==` 和 `!=` 操作符，我们想做比较！如果你试着把类型改成 `any`，

```go
func AssertNotEqual[T any](got, want T)
```

你会得到下面的错误：

```
prog.go2:15:5: cannot compare got != want (operator != not defined for T)
```

这很合理，因为你不能在每个（或者说 `any`）类型上使用这些操作符。

### 用 [`T any`](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md#the-constraint) 的泛型函数和 `interface{}` 一样吗？

考虑两个函数

```go
func GenericFoo[T any](x, y T)
```

```go
func InterfaceyFoo(x, y interface{})
```

这里泛型有什么意义？`any` 不就描述...任何东西吗？

就约束而言，`any` 确实意味着"任何东西"，`interface{}` 也是。事实上，`any` 是在 1.18 加入的，并且 _就是 `interface{}` 的别名_。

泛型版本的区别在于 _你仍然在描述一个特定的类型_，这意味着我们仍然把这个函数约束为只能与 _一种_ 类型一起工作。

这意味着你可以用任意类型组合调用 `InterfaceyFoo`（例如 `InterfaceyFoo(apple, orange)`）。然而 `GenericFoo` 仍然提供一些约束，因为我们说过它只与 _一种_ 类型 `T` 一起工作。

合法：

- `GenericFoo(apple1, apple2)`
- `GenericFoo(orange1, orange2)`
- `GenericFoo(1, 2)`
- `GenericFoo("one", "two")`

不合法（编译失败）：

- `GenericFoo(apple1, orange1)`
- `GenericFoo("1", 1)`

如果你的函数返回泛型类型，调用方也可以直接以原本的类型使用它，而不必做类型断言，因为当一个函数返回 `interface{}` 时，编译器无法对类型做出任何保证。

## 接下来：泛型数据类型

我们要创建一个[栈](https://en.wikipedia.org/wiki/Stack_(abstract_data_type))数据类型。从需求角度来说栈应该相当容易理解。它们是一组元素的集合，你可以 `Push` 元素到"顶部"，要把元素拿回来，你 `Pop` 顶部的元素（LIFO，后进先出）。

为了简洁起见，我省略了让我得到下面这段 `int` 栈和 `string` 栈代码的 TDD 过程。

```go
type StackOfInts struct {
	values []int
}

func (s *StackOfInts) Push(value int) {
	s.values = append(s.values, value)
}

func (s *StackOfInts) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfInts) Pop() (int, bool) {
	if s.IsEmpty() {
		return 0, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}

type StackOfStrings struct {
	values []string
}

func (s *StackOfStrings) Push(value string) {
	s.values = append(s.values, value)
}

func (s *StackOfStrings) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfStrings) Pop() (string, bool) {
	if s.IsEmpty() {
		return "", false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

我又创建了几个断言函数来辅助

```go
func AssertTrue(t *testing.T, got bool) {
	t.Helper()
	if !got {
		t.Errorf("got %v, want true", got)
	}
}

func AssertFalse(t *testing.T, got bool) {
	t.Helper()
	if got {
		t.Errorf("got %v, want false", got)
	}
}
```

下面是测试

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := new(StackOfInts)

		// 检查栈是空的
		AssertTrue(t, myStackOfInts.IsEmpty())

		// 加一个东西，然后检查它不空
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// 再加一个，然后弹出来
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())
	})

	t.Run("string stack", func(t *testing.T) {
		myStackOfStrings := new(StackOfStrings)

		// 检查栈是空的
		AssertTrue(t, myStackOfStrings.IsEmpty())

		// 加一个东西，然后检查它不空
		myStackOfStrings.Push("123")
		AssertFalse(t, myStackOfStrings.IsEmpty())

		// 再加一个，然后弹出来
		myStackOfStrings.Push("456")
		value, _ := myStackOfStrings.Pop()
		AssertEqual(t, value, "456")
		value, _ = myStackOfStrings.Pop()
		AssertEqual(t, value, "123")
		AssertTrue(t, myStackOfStrings.IsEmpty())
	})
}
```

### 问题

- `StackOfStrings` 和 `StackOfInts` 的代码几乎完全相同。虽然重复并不总是世界末日，但要读、写和维护的代码更多了。
- 由于我们在两个类型间复制了逻辑，我们也不得不复制测试。

我们真正想要的是用一个类型来捕获栈的 _概念_，并为其编写一套测试。我们现在应该戴上重构的帽子，这意味着我们不应该改测试，因为我们想保持相同的行为。

如果没有泛型，下面是我们 _可以_ 做的

```go
type StackOfInts = Stack
type StackOfStrings = Stack

type Stack struct {
	values []interface{}
}

func (s *Stack) Push(value interface{}) {
	s.values = append(s.values, value)
}

func (s *Stack) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack) Pop() (interface{}, bool) {
	if s.IsEmpty() {
		var zero interface{}
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

- 我们把之前 `StackOfInts` 和 `StackOfStrings` 的实现起别名指向一个新的统一类型 `Stack`
- 我们让 `values` 成为 `interface{}` 的[切片](https://github.com/quii/learn-go-with-tests/blob/main/arrays-and-slices.md)，从而去掉了 `Stack` 的类型安全

要试这段代码，你必须把我们 assert 函数上的类型约束去掉：

```go
func AssertEqual(t *testing.T, got, want interface{})
```

如果你这样做，我们的测试还是会通过。谁还需要泛型？

### 抛弃类型安全的问题

第一个问题和我们在 `AssertEquals` 中看到的一样——我们丢失了类型安全。我现在可以把苹果 `Push` 进一个橘子的栈。

即使我们有自律不这样做，这段代码用起来仍然不愉快，因为当方法 **返回 `interface{}` 时，使用起来很糟糕**。

加上下面这个测试，

```go
t.Run("interface stack DX is horrid", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()
	AssertEqual(t, firstNum+secondNum, 3)
})
```

你会得到一个编译错误，展示了丢失类型安全的弱点：

```
invalid operation: operator + not defined on firstNum (variable of type interface{})
```

当 `Pop` 返回 `interface{}` 时，意味着编译器没有关于数据是什么的任何信息，因此严重限制了我们能做什么。它无法知道这应该是一个整数，所以它不让我们使用 `+` 操作符。

要绕过这个问题，调用方必须为每个值做一次[类型断言](https://golang.org/ref/spec#Type_assertions)。

```go
t.Run("interface stack dx is horrid", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()

	// 从 interface{} 中拿出我们的 int
	reallyFirstNum, ok := firstNum.(int)
	AssertTrue(t, ok) // 需要检查我们确实从 interface{} 拿到了一个 int

	reallySecondNum, ok := secondNum.(int)
	AssertTrue(t, ok) // 又来一遍！

	AssertEqual(t, reallyFirstNum+reallySecondNum, 3)
})
```

这个测试散发出的不愉快会在我们 `Stack` 实现的每个潜在使用者身上重演，呃。

### 泛型数据结构来救场

就像你可以为函数定义泛型参数一样，你也可以定义泛型数据结构。

下面是我们新的 `Stack` 实现，使用了泛型数据类型。

```go
type Stack[T any] struct {
	values []T
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

下面是测试，展示了它们如我们所愿地工作，具有完全的类型安全。

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := new(Stack[int])

		// 检查栈是空的
		AssertTrue(t, myStackOfInts.IsEmpty())

		// 加一个东西，然后检查它不空
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// 再加一个，然后弹出来
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// 可以以数字形式拿到我们放进去的数字，而不是无类型的 interface{}
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```

你会注意到定义泛型数据结构的语法和给函数定义泛型参数的语法是一致的。

```go
type Stack[T any] struct {
	values []T
}
```

它和之前 _几乎_ 一样，只是我们说的是 **栈的类型约束了你能处理什么类型的值**。

一旦你创建了一个 `Stack[Orange]` 或 `Stack[Apple]`，定义在我们栈上的方法就只允许你传入并只会返回你正在处理的栈的特定类型：

```go
func (s *Stack[T]) Pop() (T, bool)
```

你可以想象，根据你创建的栈的类型，会以某种方式为你生成相应类型的实现：

```go
func (s *Stack[Orange]) Pop() (Orange, bool)
```

```go
func (s *Stack[Apple]) Pop() (Apple, bool)
```

既然我们做了这次重构，就可以安全地删掉字符串栈的测试，因为我们不需要一遍又一遍证明同样的逻辑。

注意，目前在调用泛型函数的例子中，我们都不需要指定泛型类型。例如，调用 `AssertEqual[T]` 时，我们不需要指定 `T` 是什么类型，因为它可以从参数推断出来。在泛型类型无法被推断的情况下，你需要在调用函数时指定类型。语法和定义函数时一样，即在参数前用方括号指定类型。

具体例子是为 `Stack[T]` 写一个构造函数。
```go
func NewStack[T any]() *Stack[T] {
	return new(Stack[T])
}
```
要用这个构造函数来分别创建一个 int 栈和一个 string 栈，你这样调用它：
```go
myStackOfInts := NewStack[int]()
myStackOfStrings := NewStack[string]()
```

下面是加了构造函数后的 `Stack` 实现和测试。

```go
type Stack[T any] struct {
	values []T
}

func NewStack[T any]() *Stack[T] {
	return new(Stack[T])
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := NewStack[int]()

		// 检查栈是空的
		AssertTrue(t, myStackOfInts.IsEmpty())

		// 加一个东西，然后检查它不空
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// 再加一个，然后弹出来
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// 可以以数字形式拿到我们放进去的数字，而不是无类型的 interface{}
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```


使用泛型数据类型后我们：

- 减少了重要逻辑的重复。
- 让 `Pop` 返回 `T`，这样如果我们创建一个 `Stack[int]`，实际上从 `Pop` 拿回来的就是 `int`；我们现在可以用 `+`，而不需要类型断言的体操。
- 在编译期防止误用。你不能把橘子 `Push` 到苹果栈里。

## 总结

本章应该让你尝到了泛型语法的滋味，并对泛型为什么有用有了一些想法。我们写了自己的 `Assert` 函数，可以放心地复用它来试验关于泛型的其他想法，并实现了一个可以以类型安全方式存储任何我们希望的数据类型的简单数据结构。

### 大多数情况下，泛型比使用 `interface{}` 更简单

如果你对静态类型语言不熟悉，泛型的意义可能不会立刻显现，但我希望本章的例子已经说明了 Go 语言在哪些地方不像我们希望的那样有表达力。特别是使用 `interface{}` 让你的代码：

- 更不安全（混入苹果和橘子），需要更多错误处理
- 表达力更弱，`interface{}` 没有告诉你任何关于数据的信息
- 更可能依赖[反射](https://github.com/quii/learn-go-with-tests/blob/main/reflection.md)、类型断言等，使你的代码更难处理且更容易出错，因为它把检查从编译期推到了运行时

使用静态类型语言是一种描述约束的行为。如果你做得好，你会写出不仅安全、易用，而且更容易写的代码，因为可能的解空间更小。

泛型给了我们一种在代码中表达约束的新方式，正如所演示的那样，它将允许我们整合并简化在 Go 1.18 之前无法做到的代码。

### 泛型会把 Go 变成 Java 吗？

- 不会。

Go 社区中关于泛型的[FUD（恐惧、不确定和怀疑）](https://en.wikipedia.org/wiki/Fear,_uncertainty,_and_doubt)很多，认为它会导致噩梦般的抽象和让人摸不着头脑的代码库。这通常会附带一句"必须小心使用"。

虽然这是真的，但这并不是特别有用的建议，因为这话对任何语言特性都成立。

并没有多少人抱怨我们能定义接口，而接口和泛型一样，是一种在我们代码中描述约束的方式。当你描述一个接口时，你正在做一个 _可能很糟糕_ 的设计选择，泛型在制造令人困惑、用起来烦人的代码方面并不独特。

### 你已经在用泛型了

考虑到如果你用过数组、切片或 map，你就 _已经是泛型代码的消费者_ 了。

```
var myApples []Apple
// 你不能这么干！
append(myApples, Orange{})
```

### 抽象不是脏话

很容易吐槽 [AbstractSingletonProxyFactoryBean](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/framework/AbstractSingletonProxyFactoryBean.html)，但我们不要假装一个完全没有抽象的代码库就不糟糕。在合适的时候 _收敛_ 相关概念是你的工作，这样你的系统更容易理解和修改；而不是一堆杂乱无章、缺乏清晰度的函数和类型。

### [先让它能跑，再让它正确，再让它快](https://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast#:~:text=%22Make%20it%20work%2C%20make%20it,to%20DesignForPerformance%20ahead%20of%20time.)

人们在没有足够信息做出好的设计决策之前就过早抽象，会遇到问题。

TDD 的红、绿、重构循环意味着你能更好地了解 _你实际需要的_ 代码，**而不是凭空想象抽象**；但你仍然需要小心。

这里没有硬性规则，但抵制住把东西做成泛型的诱惑，直到你能看到一个有用的泛化。当我们创建各种 `Stack` 实现时，重要的是我们从 _具体_ 行为开始，比如 `StackOfStrings` 和 `StackOfInts`，并以测试做支撑。从我们 _真实_ 的代码出发，我们能开始看到真正的模式，并在测试的支撑下，探索朝着更通用解决方案的重构。

人们经常建议你只在看到同样的代码三次之后才做泛化，这看起来是个不错的入门法则。

我在其他编程语言中常走的路径是：

- 一个 TDD 循环来驱动某个行为
- 另一个 TDD 循环来锻炼一些其他相关场景

> 嗯，这些东西看起来很相似——但比起耦合到糟糕的抽象，少量重复是更好的

- 睡一觉
- 又一个 TDD 循环

> 好，我想试试看能不能泛化这玩意。谢天谢地我聪明又好看，因为我用 TDD，所以我可以在任何时候重构，并且这个流程帮我在过度设计之前理解了我实际需要什么行为。

- 这个抽象感觉不错！测试还在通过，代码更简单了
- 我现在可以删掉一些测试了，我已经捕获了行为的 _本质_，去掉了不必要的细节
