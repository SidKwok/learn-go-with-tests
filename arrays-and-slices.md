# 数组和切片

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/arrays)**

数组让你可以以特定的顺序在一个变量里存储多个相同类型的元素。

当你有数组时，常常需要遍历它们。所以让我们用 [我们刚学到的 `for`](iteration.md) 知识来写一个 `Sum` 函数。`Sum` 接收一个数字数组并返回总和。

让我们运用 TDD 技能

## 先写测试

新建一个文件夹来工作。创建一个新文件叫 `sum_test.go`，并加入下面的内容：

```go
package main

import "testing"

func TestSum(t *testing.T) {

	numbers := [5]int{1, 2, 3, 4, 5}

	got := Sum(numbers)
	want := 15

	if got != want {
		t.Errorf("got %d want %d given, %v", got, want, numbers)
	}
}
```

数组在你声明变量时定义了一个 _固定容量_。
我们可以用两种方式初始化数组：

* \[N\]type{value1, value2, ..., valueN}，例如 `numbers := [5]int{1, 2, 3, 4, 5}`
* \[...\]type{value1, value2, ..., valueN}，例如 `numbers := [...]int{1, 2, 3, 4, 5}`

有时候在错误信息里把传给函数的输入也打印出来会很有用。
这里我们使用 `%v` 占位符来打印"默认"格式，对数组来说效果不错。

[阅读更多关于格式化字符串的内容](https://golang.org/pkg/fmt/)

## 尝试运行测试

如果你用 `go mod init main` 初始化了 go mod，你会看到错误
`_testmain.go:13:2: cannot import "main"`。这是因为按照常见做法，
package main 只会包含其他包的集成而不包含可单元测试的代码，
因此 Go 不会让你 import 名为 `main` 的包。

要修复这个问题，你可以把 `go.mod` 中的 main 模块名重命名为其他名字。

修复上述错误后，如果你运行 `go test`，编译器会失败，给出熟悉的
`./sum_test.go:10:15: undefined: Sum` 错误。现在我们可以继续写实际要测试的方法了。

## 写最少的代码让测试可以运行，并查看失败的测试输出

在 `sum.go` 里

```go
package main

func Sum(numbers [5]int) int {
	return 0
}
```

你的测试现在应该会失败，给出 _清晰的错误信息_

`sum_test.go:13: got 0 want 15 given, [1 2 3 4 5]`

## 写足够的代码让测试通过

```go
func Sum(numbers [5]int) int {
	sum := 0
	for i := 0; i < 5; i++ {
		sum += numbers[i]
	}
	return sum
}
```

要从数组的某个特定索引取值，只需使用 `array[index]`
语法。这里我们用 `for` 迭代 5 次，遍历数组并把每个元素加到 `sum` 上。

## 重构

让我们引入 [`range`](https://gobyexample.com/range) 来帮助清理代码

```go
func Sum(numbers [5]int) int {
	sum := 0
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

`range` 让你可以迭代一个数组。每次迭代，`range` 会返回两个值——索引和值。
我们选择用 `_` [空白标识符](https://golang.org/doc/effective_go.html#blank) 来忽略索引值。

### 数组及其类型

数组的一个有趣特性是大小被编码在它的类型里。如果你试图把一个 `[4]int`
传入一个期望 `[5]int` 的函数，它不会编译通过。
它们是不同的类型，就像试图把 `string` 传入一个想要 `int` 的函数一样。

你可能会觉得数组有固定长度有点麻烦，大多数时候你应该不会用到它们！

Go 有 _切片_，它不在类型里编码集合的大小，而是可以是任意大小。

下一个需求是对不同大小的集合求和。

## 先写测试

我们现在使用 [切片类型][slice]，它让我们可以拥有任意大小的集合。语法和数组非常相似，
只是声明时省略了大小

`mySlice := []int{1,2,3}` 而不是 `myArray := [3]int{1,2,3}`

```go
func TestSum(t *testing.T) {

	t.Run("collection of 5 numbers", func(t *testing.T) {
		numbers := [5]int{1, 2, 3, 4, 5}

		got := Sum(numbers)
		want := 15

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

	t.Run("collection of any size", func(t *testing.T) {
		numbers := []int{1, 2, 3}

		got := Sum(numbers)
		want := 6

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

}
```

## 尝试运行测试

这编译不通过

`./sum_test.go:22:13: cannot use numbers (type []int) as type [5]int in argument to Sum`

## 写最少的代码让测试可以运行，并查看失败的测试输出

这里的问题是我们要么

* 通过把 `Sum` 的参数从数组改成切片来打破现有的 API。这样做的话，
  我们可能会毁了某人的一天，因为我们的 _另一个_ 测试不再能编译！
* 创建一个新函数

在我们的例子里，没有别人在用我们的函数，所以与其维护两个函数，不如就用一个。

```go
func Sum(numbers []int) int {
	sum := 0
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

如果你尝试运行测试，它们仍然不会编译通过，你需要把第一个测试改成传入切片而不是数组。

## 写足够的代码让测试通过

事实证明，修复编译器问题就是我们这里要做的全部，测试通过了！

## 重构

我们已经重构了 `Sum`——我们所做的就是把数组替换成切片，所以不需要更多改动。
记住，重构阶段我们不能忽视测试代码——我们可以进一步改进 `Sum` 测试。

```go
func TestSum(t *testing.T) {

	t.Run("collection of 5 numbers", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5}

		got := Sum(numbers)
		want := 15

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

	t.Run("collection of any size", func(t *testing.T) {
		numbers := []int{1, 2, 3}

		got := Sum(numbers)
		want := 6

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

}
```

质疑测试的价值很重要。目标不应是拥有尽可能多的测试，而是对你的代码库有尽可能多的 _信心_。
测试太多会成为一个真正的问题，会增加更多维护开销。**每个测试都有成本**。

在我们的例子里，你可以看到为这个函数有两个测试是冗余的。
如果它对一个大小的切片有效，那么它对任意大小的切片也很可能有效（在合理范围内）。

Go 内置的测试工具集有一个 [覆盖率工具](https://blog.golang.org/cover)。
虽然追求 100% 的覆盖率不应是你的最终目标，但覆盖率工具可以帮助
找出代码中没有被测试覆盖的区域。如果你严格遵循 TDD，
你的覆盖率很可能也接近 100%。

试着运行

`go test -cover`

你应该会看到

```bash
PASS
coverage: 100.0% of statements
```

现在删除其中一个测试，再检查一下覆盖率。

既然我们对一个测试良好的函数感到满意，你应该在迎接下一个挑战之前
提交你的优秀工作。

我们需要一个新函数 `SumAll`，它接收变化数量的切片，
返回一个新切片，其中包含每个传入切片的总和。

例如

`SumAll([]int{1,2}, []int{0,9})` 会返回 `[]int{3, 9}`

或者

`SumAll([]int{1,1,1})` 会返回 `[]int{3}`

## 先写测试

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := []int{3, 9}

	if got != want {
		t.Errorf("got %v want %v", got, want)
	}
}
```

## 尝试运行测试

`./sum_test.go:23:9: undefined: SumAll`

## 写最少的代码让测试可以运行，并查看失败的测试输出

我们需要按照测试想要的方式定义 `SumAll`。

Go 让你可以写 [_可变参数函数_](https://gobyexample.com/variadic-functions)，它们可以接收可变数量的参数。

```go
func SumAll(numbersToSum ...[]int) []int {
	return nil
}
```

这是合法的，但我们的测试还是编译不通过！

`./sum_test.go:26:9: invalid operation: got != want (slice can only be compared to nil)`

Go 不允许你对切片使用相等运算符。你 _可以_ 写一个函数迭代每个 `got` 和 `want` 切片并检查它们的值，
但如果有更方便的办法呢？

从 Go 1.21 开始，[slices](https://pkg.go.dev/slices#pkg-overview) 标准包可用，它有 [slices.Equal](https://pkg.go.dev/slices#Equal) 函数对切片进行简单的浅比较，你不必担心像上面这样的类型问题。
注意这个函数要求元素是 [comparable](https://pkg.go.dev/builtin#comparable) 的。
所以它不能用于元素不可比较的切片，比如二维切片。

让我们把这个付诸实践！

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := []int{3, 9}

	if !slices.Equal(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

你应该会得到类似下面这样的测试输出：
`sum_test.go:30: got [] want [3 9]`

## 写足够的代码让测试通过

我们要做的是迭代可变参数，使用现有的 `Sum` 函数计算总和，
然后把它加到我们将要返回的切片里

```go
func SumAll(numbersToSum ...[]int) []int {
	lengthOfNumbers := len(numbersToSum)
	sums := make([]int, lengthOfNumbers)

	for i, numbers := range numbersToSum {
		sums[i] = Sum(numbers)
	}

	return sums
}
```

要学的新东西很多！

有了一种创建切片的新方式。`make` 让你可以创建一个起始容量为 `numbersToSum` 长度的切片。
切片的长度是它持有的元素数量 `len(mySlice)`，而容量是它在底层数组中能持有的元素数量 `cap(mySlice)`，
例如 `make([]int, 0, 5)` 创建一个长度为 0、容量为 5 的切片。

你可以像数组一样用 `mySlice[N]` 索引切片来取值，
或者用 `=` 给它赋一个新值

测试现在应该能通过了。

## 重构

正如所述，切片有容量。如果你有一个容量为 2 的切片，
然后尝试 `mySlice[10] = 1`，你会得到一个 _运行时_ 错误。

但是，你可以使用 `append` 函数，它接收一个切片和一个新值，
然后返回一个包含所有元素的新切片。

```go
func SumAll(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		sums = append(sums, Sum(numbers))
	}

	return sums
}
```

在这个实现中，我们对容量的关注更少了。我们从一个空切片 `sums` 开始，
随着我们处理可变参数，把 `Sum` 的结果追加到它上面。

我们的下一个需求是把 `SumAll` 改为 `SumAllTails`，它会
计算每个切片"尾部"的总和。集合的尾部是
集合中除第一个元素（"头部"）外的所有元素。

## 先写测试

```go
func TestSumAllTails(t *testing.T) {
	got := SumAllTails([]int{1, 2}, []int{0, 9})
	want := []int{2, 9}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

## 尝试运行测试

`./sum_test.go:26:9: undefined: SumAllTails`

## 写最少的代码让测试可以运行，并查看失败的测试输出

把函数重命名为 `SumAllTails` 并重新运行测试

`sum_test.go:30: got [3 9] want [2 9]`

## 写足够的代码让测试通过

```go
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		tail := numbers[1:]
		sums = append(sums, Sum(tail))
	}

	return sums
}
```

切片可以被切片！语法是 `slice[low:high]`。如果你省略 `:` 一侧的值，
它会捕获那一侧的所有内容。在我们的例子里，
`numbers[1:]` 表示"从 1 取到末尾"。你可能想花一些时间
针对切片写其他测试，并尝试切片操作符以更熟悉它。

## 重构

这次没什么要重构的。

如果你给我们的函数传入一个空切片，会发生什么？空切片的"尾部"是什么？
当你让 Go 从 `myEmptySlice[1:]` 捕获所有元素时会怎样？

## 先写测试

```go
func TestSumAllTails(t *testing.T) {

	t.Run("make the sums of some slices", func(t *testing.T) {
		got := SumAllTails([]int{1, 2}, []int{0, 9})
		want := []int{2, 9}

		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	})

	t.Run("safely sum empty slices", func(t *testing.T) {
		got := SumAllTails([]int{}, []int{3, 4, 5})
		want := []int{0, 9}

		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	})

}
```

## 尝试运行测试

```text
panic: runtime error: slice bounds out of range [recovered]
    panic: runtime error: slice bounds out of range
```

哦不！要注意，虽然测试 _编译通过了_，但 _有一个运行时错误_。

编译时错误是我们的朋友，因为它们帮我们写出能工作的软件，
而运行时错误是我们的敌人，因为它们影响我们的用户。

## 写足够的代码让测试通过

```go
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

## 重构

我们的测试在断言部分又有重复代码了，让我们把它们抽成一个函数。

```go
func TestSumAllTails(t *testing.T) {

	checkSums := func(t testing.TB, got, want []int) {
		t.Helper()
		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	}

	t.Run("make the sums of tails of", func(t *testing.T) {
		got := SumAllTails([]int{1, 2}, []int{0, 9})
		want := []int{2, 9}
		checkSums(t, got, want)
	})

	t.Run("safely sum empty slices", func(t *testing.T) {
		got := SumAllTails([]int{}, []int{3, 4, 5})
		want := []int{0, 9}
		checkSums(t, got, want)
	})

}
```

我们本可以像往常一样创建一个新函数 `checkSums`，但在这里，我们展示了一种新技术：把一个函数赋给变量。这看起来可能有点奇怪，但和把变量赋给 `string` 或 `int` 没有区别，函数实际上也是值。

这里没有展示，但当你想把一个函数绑定到"作用域"内的其他局部变量时（例如某些 `{}` 之间），这种技术很有用。它还允许你减少 API 的暴露面。

通过把这个函数定义在测试内部，它就不能被这个包内的其他函数使用。把不需要导出的变量和函数隐藏起来是一个重要的设计考量。

它的一个方便的副作用是给我们的代码增加了一点类型安全。如果一个开发者错误地添加了一个新测试 `checkSums(t, got, "dave")`，编译器会立刻拦住他们。

```bash
$ go test
./sum_test.go:52:21: cannot use "dave" (type string) as type []int in argument to checkSums
```

## 总结

我们已经覆盖了

* 数组
* 切片
  * 创建它们的多种方式
  * 它们如何有 _固定_ 容量，但你可以用 `append` 从旧切片创建新切片
  * 如何切片，切片！
* `len` 用来获取数组或切片的长度
* 测试覆盖率工具
* `reflect.DeepEqual` 以及它为什么有用，但会降低代码的类型安全性

我们用整数演示了切片和数组，但它们也适用于其他任何类型，
包括数组/切片本身。所以如果你需要，可以声明一个
`[][]string` 类型的变量。

[查看 Go 博客关于切片的文章][blog-slice]，深入了解切片。
通过写更多测试来巩固你从中学到的知识。

除了写测试，另一种动手尝试 Go 的便利方式是 Go playground。
你可以试大多数东西，并且如果你需要提问，可以方便地分享代码。
[我做了一个 Go playground，里面有一个切片供你尝试](https://play.golang.org/p/ICCWcRGIO68)。

[这是一个例子](https://play.golang.org/p/bTrRmYfNYCp) 演示了对数组的切片操作
以及修改切片如何影响原数组；但切片的"副本"
不会影响原数组。
[另一个例子](https://play.golang.org/p/Poth8JS28sc) 说明了为什么
对一个非常大的切片进行切片后再做副本是个好主意。

[for]: ../iteration.md#
[blog-slice]: https://blog.golang.org/go-slices-usage-and-internals
[deepEqual]: https://golang.org/pkg/reflect/#DeepEqual
[slice]: https://golang.org/doc/effective_go.html#slices
