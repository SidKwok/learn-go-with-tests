# 结构体、方法和接口

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/structs)**

假设我们需要一些几何代码，根据高度和宽度来计算矩形的周长。我们可以写一个 `Perimeter(width float64, height float64)` 函数，其中 `float64` 是用来表示像 `123.45` 这样的浮点数。

到现在为止，TDD 循环对你应该已经很熟悉了。

## 先写测试

```go
func TestPerimeter(t *testing.T) {
	got := Perimeter(10.0, 10.0)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

注意到这个新的格式化字符串了吗？`f` 是给我们的 `float64` 用的，`.2` 表示打印 2 位小数。

## 尝试运行测试

`./shapes_test.go:6:9: undefined: Perimeter`

## 写最少的代码让测试可以运行，并查看失败的测试输出

```go
func Perimeter(width float64, height float64) float64 {
	return 0
}
```

会得到 `shapes_test.go:10: got 0.00 want 40.00`。

## 写足够的代码让测试通过

```go
func Perimeter(width float64, height float64) float64 {
	return 2 * (width + height)
}
```

到目前为止还挺简单。现在让我们创建一个名为 `Area(width, height float64)` 的函数，它返回矩形的面积。

按照 TDD 循环，自己试着做一下。

你应该会写出像下面这样的测试

```go
func TestPerimeter(t *testing.T) {
	got := Perimeter(10.0, 10.0)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}

func TestArea(t *testing.T) {
	got := Area(12.0, 6.0)
	want := 72.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

以及像下面这样的代码

```go
func Perimeter(width float64, height float64) float64 {
	return 2 * (width + height)
}

func Area(width float64, height float64) float64 {
	return width * height
}
```

## 重构

我们的代码完成了任务，但里面没有任何东西明确地表示这是关于矩形的。一个不小心的开发者可能会试着把三角形的宽和高传给这些函数，却没意识到它们会返回错误的答案。

我们可以给函数起更具体的名字，比如 `RectangleArea`。一个更优雅的解法是定义一个我们自己的 _类型_ 叫 `Rectangle`，把这个概念封装起来。

我们可以使用 **结构体** 来创建一个简单的类型。[结构体](https://golang.org/ref/spec#Struct_types) 就是一个有名字的字段集合，你可以在里面存储数据。

在你的 `shapes.go` 文件里像下面这样声明一个结构体

```go
type Rectangle struct {
	Width  float64
	Height float64
}
```

现在让我们重构测试，使用 `Rectangle` 而不是普通的 `float64`。

```go
func TestPerimeter(t *testing.T) {
	rectangle := Rectangle{10.0, 10.0}
	got := Perimeter(rectangle)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}

func TestArea(t *testing.T) {
	rectangle := Rectangle{12.0, 6.0}
	got := Area(rectangle)
	want := 72.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

记得在尝试修复之前先运行测试。测试应该会显示一个有用的错误，比如

```text
./shapes_test.go:7:18: not enough arguments in call to Perimeter
    have (Rectangle)
    want (float64, float64)
```

你可以通过 `myStruct.field` 这样的语法来访问结构体的字段。

修改这两个函数让测试通过。

```go
func Perimeter(rectangle Rectangle) float64 {
	return 2 * (rectangle.Width + rectangle.Height)
}

func Area(rectangle Rectangle) float64 {
	return rectangle.Width * rectangle.Height
}
```

我希望你会同意，给函数传入一个 `Rectangle` 能更清楚地表达我们的意图，但使用结构体还有更多好处，我们之后会讲到。

我们的下一个需求是为圆写一个 `Area` 函数。

## 先写测试

```go
func TestArea(t *testing.T) {

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		got := Area(rectangle)
		want := 72.0

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		got := Area(circle)
		want := 314.1592653589793

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

}
```

如你所见，`f` 被替换成了 `g`，这是有原因的。
使用 `g` 在错误消息里会打印出更精确的小数 \([fmt 选项](https://golang.org/pkg/fmt/)\)。
例如，在圆的面积计算中使用半径 1.5，`f` 会显示 `7.068583`，而 `g` 会显示 `7.0685834705770345`。

## 尝试运行测试

`./shapes_test.go:28:13: undefined: Circle`

## 写最少的代码让测试可以运行，并查看失败的测试输出

我们需要定义 `Circle` 类型。

```go
type Circle struct {
	Radius float64
}
```

现在再试着运行测试

`./shapes_test.go:29:14: cannot use circle (type Circle) as type Rectangle in argument to Area`

有些编程语言允许你这样做：

```go
func Area(circle Circle) float64       {}
func Area(rectangle Rectangle) float64 {}
```

但在 Go 里你不能这样做

`./shapes.go:20:32: Area redeclared in this block`

我们有两个选择：

* 你可以在不同的 _包_ 里声明同名函数。所以我们可以在一个新包里创建 `Area(Circle)`，但这里这么做感觉小题大做。
* 我们可以在新定义的类型上定义 [_方法_](https://golang.org/ref/spec#Method_declarations)。

### 什么是方法？

到目前为止我们只写过 _函数_，但我们一直在使用一些方法。当我们调用 `t.Errorf` 时，我们是在 `t` （`testing.T`）的实例上调用 `Errorf` 方法。

方法是带有接收器的函数。
方法声明把一个标识符（方法名）绑定到一个方法，并把方法和接收器的基础类型关联起来。

方法和函数非常相似，但它们要在某个特定类型的实例上调用。函数你可以随便在哪里调用，比如 `Area(rectangle)`，但方法只能在"东西"上调用。

举个例子能更直观，所以让我们先修改测试以调用方法，然后再修代码。

```go
func TestArea(t *testing.T) {

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		got := rectangle.Area()
		want := 72.0

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		got := circle.Area()
		want := 314.1592653589793

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

}
```

如果我们尝试运行测试，会得到

```text
./shapes_test.go:19:19: rectangle.Area undefined (type Rectangle has no field or method Area)
./shapes_test.go:29:16: circle.Area undefined (type Circle has no field or method Area)
```

> type Circle has no field or method Area

我想再次强调编译器在这里有多棒。慢慢花时间读你拿到的错误信息真的很重要，长远来看会大有帮助。

## 写最少的代码让测试可以运行，并查看失败的测试输出

让我们给类型加一些方法

```go
type Rectangle struct {
	Width  float64
	Height float64
}

func (r Rectangle) Area() float64 {
	return 0
}

type Circle struct {
	Radius float64
}

func (c Circle) Area() float64 {
	return 0
}
```

声明方法的语法和函数几乎相同，因为它们非常相似。唯一的区别是方法接收器的语法 `func (receiverName ReceiverType) MethodName(args)`。

当方法在某个类型的变量上被调用时，你通过 `receiverName` 变量获得对其数据的引用。在很多其他编程语言里，这是隐式做的，你通过 `this` 来访问接收器。

在 Go 中，按惯例接收器变量取类型名的首字母。

```
r Rectangle
```

如果你再次运行测试，它们现在应该能编译了，并给你一些失败输出。

## 写足够的代码让测试通过

现在我们通过修复新方法来让矩形的测试通过

```go
func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}
```

如果你重新运行测试，矩形的测试应该能通过了，但圆的还在失败。

要让圆的 `Area` 函数通过，我们会借用 `math` 包里的 `Pi` 常量（记得 import 它）。

```go
func (c Circle) Area() float64 {
	return math.Pi * c.Radius * c.Radius
}
```

## 重构

我们的测试里有些重复。

我们想做的只是拿到一组 _形状_，对它们调用 `Area()` 方法，然后检查结果。

我们希望能写一种 `checkArea` 函数，可以把 `Rectangle` 和 `Circle` 都传给它，但如果传入了不是形状的东西就编译失败。

在 Go 中，我们可以用 **接口** 把这个意图编码进来。

[接口](https://golang.org/ref/spec#Interface_types) 在像 Go 这样的静态类型语言里是一个非常强大的概念，因为它让你可以写出能用于不同类型的函数，并创建高度解耦的代码，同时仍然保持类型安全。

我们通过重构测试来介绍这一点。

```go
func TestArea(t *testing.T) {

	checkArea := func(t testing.TB, shape Shape, want float64) {
		t.Helper()
		got := shape.Area()
		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	}

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		checkArea(t, rectangle, 72.0)
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		checkArea(t, circle, 314.1592653589793)
	})

}
```

我们像在其他练习里那样创建了一个辅助函数，但这次我们要求传入一个 `Shape`。如果我们尝试用一个不是形状的东西调用它，它就不会通过编译。

一个东西怎样才能成为形状？我们只需用接口声明告诉 Go `Shape` 是什么

```go
type Shape interface {
	Area() float64
}
```

我们正在创建一个新的 `type`，就像我们对 `Rectangle` 和 `Circle` 所做的那样，但这次它是 `interface` 而不是 `struct`。

一旦你把这个加入代码，测试就会通过。

### 等等，啥？

这和大多数其他编程语言中的接口非常不一样。通常你需要写代码声明 `My type Foo implements interface Bar`。

但在我们的例子里

* `Rectangle` 有一个名为 `Area` 的方法返回 `float64`，所以它满足 `Shape` 接口
* `Circle` 有一个名为 `Area` 的方法返回 `float64`，所以它满足 `Shape` 接口
* `string` 没有这样的方法，所以它不满足这个接口
* 等等。

在 Go 中 **接口的解析是隐式的**。如果你传入的类型符合接口的要求，它就会通过编译。

### 解耦

注意我们的辅助函数不需要关心形状是 `Rectangle`、`Circle` 还是 `Triangle`。通过声明一个接口，辅助函数从具体类型中 _解耦_，只持有它做事所需的方法。

这种用接口来声明 **只声明你需要的内容** 的方法在软件设计中非常重要，后面的章节会有更详细的介绍。

## 进一步重构

现在你对结构体有了一些理解，我们可以引入"表驱动测试"了。

[表驱动测试](https://go.dev/wiki/TableDrivenTests) 在你想构建一组以同样方式测试的测试用例时非常有用。

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		shape Shape
		want  float64
	}{
		{Rectangle{12, 6}, 72.0},
		{Circle{10}, 314.1592653589793},
	}

	for _, tt := range areaTests {
		got := tt.shape.Area()
		if got != tt.want {
			t.Errorf("got %g want %g", got, tt.want)
		}
	}

}
```

这里唯一的新语法是创建一个"匿名结构体" `areaTests`。我们用 `[]struct` 声明了一个有两个字段（`shape` 和 `want`）的结构体切片。然后我们把测试用例填进切片。

接下来我们像迭代其他切片一样迭代它，使用结构体字段来运行测试。

你可以看到，开发者要引入一个新形状、实现 `Area`，再把它加到测试用例里会非常容易。此外，如果在 `Area` 中发现了 bug，加一个新的测试用例来重现它也很容易，然后再去修复。

表驱动测试可以是你工具箱里的一大利器，但要确认你确实需要测试中那些额外的"噪声"。
当你希望测试一个接口的多种实现，或者传入函数的数据有许多需要测试的不同要求时，它非常合适。

让我们通过加入另一个形状并测试它来演示这一切；一个三角形。

## 先写测试

为我们的新形状添加一个新测试很简单。只需把 `{Triangle{12, 6}, 36.0},` 加到列表中。

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		shape Shape
		want  float64
	}{
		{Rectangle{12, 6}, 72.0},
		{Circle{10}, 314.1592653589793},
		{Triangle{12, 6}, 36.0},
	}

	for _, tt := range areaTests {
		got := tt.shape.Area()
		if got != tt.want {
			t.Errorf("got %g want %g", got, tt.want)
		}
	}

}
```

## 尝试运行测试

记住，反复尝试运行测试，让编译器引导你找到解决方案。

## 写最少的代码让测试可以运行，并查看失败的测试输出

`./shapes_test.go:25:4: undefined: Triangle`

我们还没定义 `Triangle`

```go
type Triangle struct {
	Base   float64
	Height float64
}
```

再试一次

```text
./shapes_test.go:25:8: cannot use Triangle literal (type Triangle) as type Shape in field value:
    Triangle does not implement Shape (missing Area method)
```

它告诉我们不能把 `Triangle` 当作形状来用，因为它没有 `Area()` 方法，所以加一个空实现让测试能跑起来

```go
func (t Triangle) Area() float64 {
	return 0
}
```

最后代码编译通过，我们得到了错误

`shapes_test.go:31: got 0.00 want 36.00`

## 写足够的代码让测试通过

```go
func (t Triangle) Area() float64 {
	return (t.Base * t.Height) * 0.5
}
```

我们的测试通过了！

## 重构

同样，实现已经不错了，但我们的测试可以再改进一下。

当你扫一眼这个

```
{Rectangle{12, 6}, 72.0},
{Circle{10}, 314.1592653589793},
{Triangle{12, 6}, 36.0},
```

并不能立刻看清这些数字都代表什么，你应该让你的测试容易理解。

到现在为止，你看到的创建结构体实例的语法只有 `MyStruct{val1, val2}`，但你也可以选择给字段命名。

我们看看它长什么样

```
        {shape: Rectangle{Width: 12, Height: 6}, want: 72.0},
        {shape: Circle{Radius: 10}, want: 314.1592653589793},
        {shape: Triangle{Base: 12, Height: 6}, want: 36.0},
```

在 [Test-Driven Development by Example](https://g.co/kgs/yCzDLF) 中，Kent Beck 把一些测试重构到某个程度后断言：

> 测试更清晰地与我们对话，仿佛它是对真理的断言，**而不是一系列操作**

\(引用中的强调是我加的\)

现在我们的测试——更准确地说是测试用例列表——对形状及其面积做出了真理性的断言。

## 让你的测试输出有用

还记得早些时候我们实现 `Triangle` 时遇到的失败测试吗？它打印了 `shapes_test.go:31: got 0.00 want 36.00`。

我们知道这与 `Triangle` 有关，因为我们当时正在处理它。
但如果一个 bug 潜入了表里 20 个测试用例中的一个呢？
开发者怎么知道是哪个用例失败了？
这对开发者来说体验很糟糕，他们不得不手动翻看用例去找出究竟是哪个用例失败了。

我们可以把错误信息改成 `%#v got %g want %g`。`%#v` 格式化字符串会打印出我们的结构体以及它字段中的值，所以开发者能一眼看到正在测试的属性。

为进一步提升测试用例的可读性，我们可以把 `want` 字段重命名成更有描述性的，比如 `hasArea`。

关于表驱动测试的最后一个小贴士是用 `t.Run` 给测试用例命名。

通过用 `t.Run` 包装每个用例，失败时你会得到更清晰的测试输出，因为它会打印用例的名字

```text
--- FAIL: TestArea (0.00s)
    --- FAIL: TestArea/Rectangle (0.00s)
        shapes_test.go:33: main.Rectangle{Width:12, Height:6} got 72.00 want 72.10
```

并且你可以用 `go test -run TestArea/Rectangle` 来运行表中特定的测试。

下面是包含这些改进的最终测试代码

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		name    string
		shape   Shape
		hasArea float64
	}{
		{name: "Rectangle", shape: Rectangle{Width: 12, Height: 6}, hasArea: 72.0},
		{name: "Circle", shape: Circle{Radius: 10}, hasArea: 314.1592653589793},
		{name: "Triangle", shape: Triangle{Base: 12, Height: 6}, hasArea: 36.0},
	}

	for _, tt := range areaTests {
		// 使用用例中的 tt.name 作为 `t.Run` 的测试名
		t.Run(tt.name, func(t *testing.T) {
			got := tt.shape.Area()
			if got != tt.hasArea {
				t.Errorf("%#v got %g want %g", tt.shape, got, tt.hasArea)
			}
		})

	}

}
```

## 总结

这又是一次 TDD 的练习，对基础数学问题的解法进行迭代，并由测试驱动学习新的语言特性。

* 声明结构体来创建你自己的数据类型，让你可以把相关的数据捆绑在一起，并让代码意图更清晰
* 声明接口让你定义可以被不同类型使用的函数 \([临时多态](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism)\)
* 添加方法让你可以给数据类型添加功能，并让你可以实现接口
* 表驱动测试让你的断言更清晰，让测试套件更易扩展和维护

这是重要的一章，因为我们现在开始定义自己的类型。在像 Go 这样的静态类型语言中，能够设计自己的类型，对于构建易于理解、组合和测试的软件至关重要。

接口是一个把复杂性从系统其他部分隐藏起来的好工具。在我们的例子中，测试辅助 _代码_ 不需要知道它正在断言的具体形状是什么，只需要知道如何"问"它的面积。

随着你对 Go 越来越熟悉，你将开始看到接口和标准库的真正威力。你会了解到标准库中定义的接口被 _到处_ 使用，通过为自己的类型实现它们，你可以非常快地复用许多优秀的功能。
