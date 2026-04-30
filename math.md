# 数学

[**本章的所有代码可以在这里找到**](https://github.com/quii/learn-go-with-tests/tree/main/math)

尽管现代计算机能以闪电般的速度执行庞大的运算，普通开发者在工作中却很少用到任何数学。但今天不一样！今天我们要用数学来解决一个 _真实_ 的问题。而且不是无聊的数学——我们将用到三角函数、向量等等你曾发誓高中毕业后再也用不到的东西。

## 问题

你想做一个时钟的 SVG。不是数字时钟——不，那太简单了——而是一个带指针的 _模拟_ 时钟。你不需要什么花哨的东西，只需要一个不错的函数，它接收 `time` 包中的 `Time`，输出一个时钟的 SVG，所有指针——时针、分针、秒针——都指向正确的方向。能有多难？

首先我们需要一个时钟的 SVG 来做练习。SVG 是一种很棒的图像格式，便于以编程方式操作，因为它由一系列形状以 XML 描述。所以这个时钟：

![an svg of a clock](.gitbook/assets/example_clock.svg)

是这样描述的：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg"
     width="100%"
     height="100%"
     viewBox="0 0 300 300"
     version="2.0">

  <!-- bezel -->
  <circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/>

  <!-- hour hand -->
  <line x1="150" y1="150" x2="114.150000" y2="132.260000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>

  <!-- minute hand -->
  <line x1="150" y1="150" x2="101.290000" y2="99.730000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>

  <!-- second hand -->
  <line x1="150" y1="150" x2="77.190000" y2="202.900000"
        style="fill:none;stroke:#f00;stroke-width:3px;"/>
</svg>
```

它是一个圆和三条线，每条线从圆心 (x=150, y=150) 出发，延伸到某个距离之外。

所以我们要做的就是以某种方式重建上面的内容，但要根据给定的时间，让线条指向相应的方向。

## 一个验收测试

在我们陷得太深之前，先想想验收测试。

等等，你还不知道什么是验收测试。听着，让我试着解释。

我来问你：胜利是什么样子的？我们怎么知道工作完成了？TDD 提供了一种很好的方式来知道你是否完成了：当测试通过时。有时候——其实，几乎所有时候——写一个测试来告诉你整个可用功能是否完成是很不错的。不只是一个告诉你某个特定函数按你期望工作的测试，而是一个告诉你你想要实现的整件事——"功能"——是否完整的测试。

这些测试有时被称为"验收测试"，有时被称为"功能测试"。其想法是：你写一个非常高层次的测试来描述你想实现的目标——比如，一个用户点击网站上的按钮，然后看到一份他们捕获到的所有宝可梦的完整列表。一旦写好了那个测试，我们就可以再写更多测试——单元测试——来逐步搭建出一个能让验收测试通过的可工作系统。所以对我们这个例子来说，这些测试可能涉及渲染一个带按钮的网页、测试 web 服务器上的路由 handler、执行数据库查询，等等。所有这些都会用 TDD 来做，所有这些都会让最初那个验收测试通过。

类似 Nat Pryce 和 Steve Freeman 的这张 _经典_ 图

![Outside-in feedback loops in TDD](.gitbook/assets/TDD-outside-in.jpg)

总之，让我们试着写出那个验收测试——那个会告诉我们什么时候完成的测试。

我们已经有了一个示例时钟，那让我们想想哪些是重要的参数。

```
<line x1="150" y1="150" x2="114.150000" y2="132.260000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>
```

时钟的中心（这条线的 `x1` 和 `y1` 属性）对每根指针都是一样的。每根指针需要变化的数字——也就是构建 SVG 所需的参数——是 `x2` 和 `y2` 属性。我们需要为每根指针提供 X 和 Y。

我 _可以_ 考虑更多参数——表盘圆的半径、SVG 的尺寸、指针的颜色、形状，等等……但更好的做法是先用简单、具体的方案解决一个简单、具体的问题，然后再开始添加参数让它变得通用。

所以我们规定：

* 每个时钟的中心是 (150, 150)
* 时针长 50
* 分针长 80
* 秒针长 90

关于 SVG 有一点要注意：原点——点 (0,0)——在 _左上角_，而不是我们可能期望的 _左下角_。在我们计算往线条里填什么数字时，记住这一点很重要。

最后，我没有决定 _怎么_ 构造 SVG——我们可以用 [`text/template`](https://golang.org/pkg/text/template/) 包的模板，或者只是把字节发送到 `bytes.Buffer` 或一个 writer。但我们知道我们会需要这些数字，所以让我们专注于测试一些能产出这些数字的东西。

### 先写测试

所以我的第一个测试看起来是这样：

```go
package clockface_test

import (
	"projectpath/clockface"
	"testing"
	"time"
)

func TestSecondHandAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 150 - 90}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

记得 SVG 是从左上角开始绘制坐标的吗？要把秒针放在午夜位置，我们期望它在 X 轴上没有从表盘中心移动——还是 150——而 Y 轴是指针长度从中心向"上"延伸；150 减 90。

### 尝试运行测试

这驱动出我们对缺失的函数和类型的预期失败：

```
--- FAIL: TestSecondHandAtMidnight (0.00s)
./clockface_test.go:13:10: undefined: clockface.Point
./clockface_test.go:14:9: undefined: clockface.SecondHand
```

所以我们需要一个 `Point` 表示秒针尖端应该到达的位置，还需要一个函数来获取它。

### 写最少量的代码让测试运行起来，并检查失败的测试输出

让我们实现这些类型让代码能编译

```go
package clockface

import "time"

// A Point represents a two-dimensional Cartesian coordinate
type Point struct {
	X float64
	Y float64
}

// SecondHand is the unit vector of the second hand of an analogue clock at time `t`
// represented as a Point.
func SecondHand(t time.Time) Point {
	return Point{}
}
```

现在我们得到：

```
--- FAIL: TestSecondHandAtMidnight (0.00s)
    clockface_test.go:17: Got {0 0}, wanted {150 60}
FAIL
exit status 1
FAIL	learn-go-with-tests/math/clockface	0.006s
```

### 写足够的代码让它通过

当我们得到了预期的失败之后，可以填充 `SecondHand` 的返回值：

```go
// SecondHand is the unit vector of the second hand of an analogue clock at time `t`
// represented as a Point.
func SecondHand(t time.Time) Point {
	return Point{150, 60}
}
```

瞧，一个通过的测试。

```
PASS
ok  	    clockface	0.006s
```

### 重构

还不需要重构——代码刚刚好够用！

### 为新需求重复以上流程

我们大概需要做点不只是返回一个永远显示午夜的时钟的工作……

### 先写测试

```go
func TestSecondHandAt30Seconds(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 30, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 150 + 90}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

思路相同，只是现在秒针 _向下_ 指，所以我们把长度 _加_ 到 Y 轴上。

这能编译……但我们怎么让它通过？

## 思考时间

我们要怎么解决这个问题？

每分钟秒针都会经过相同的 60 个状态，指向 60 个不同的方向。当是 0 秒时它指向表盘顶部，当是 30 秒时它指向表盘底部。够简单。

那如果我想知道，比如 37 秒时秒针指向哪个方向，我会想知道 12 点钟方向和绕圆周 37/60 处之间的角度。换算成度数是 `(360 / 60 ) * 37 = 222`，但更容易记住的是它就是一整圈的 `37/60`。

但角度只是故事的一半；我们需要知道秒针尖端指向的 X 和 Y 坐标。怎么算出来？

## 数学

想象一个绕原点（坐标 `0, 0`）画的半径为 1 的圆。

![picture of the unit circle](.gitbook/assets/unit_circle.png)

这叫做"单位圆"，因为……嗯，半径是 1 个单位！

圆周由网格上的点构成——也就是更多坐标。这些坐标的 x 和 y 分量构成了一个三角形，其斜边总是 1（即圆的半径）。

![picture of the unit circle with a point defined on the circumference](.gitbook/assets/unit_circle_coords.png)

现在，三角函数可以让我们在已知它们与原点构成的角度时，算出每个三角形的 X 和 Y 长度。X 坐标会是 cos(a)，Y 坐标会是 sin(a)，其中 a 是直线与（正）X 轴的夹角。

![picture of the unit circle with the x and y elements of a ray defined as cos(a) and sin(a) respectively, where a is the angle made by the ray with the x axis](<.gitbook/assets/unit_circle_params (1).png>)

（如果你不信，[去看维基百科吧……](https://en.wikipedia.org/wiki/Sine#Unit_circle_definition)）

最后还有一个转折——因为我们想从 12 点钟方向（而不是 X 轴/3 点钟方向）开始测量角度，我们需要把坐标轴交换一下；现在 x = sin(a)，y = cos(a)。

![unit circle ray defined from by angle from y axis](.gitbook/assets/unit_circle_12_oclock.png)

所以现在我们知道怎么得到秒针的角度（每秒是圆的 1/60）以及 X、Y 坐标。我们需要 `sin` 和 `cos` 这两个函数。

## `math`

幸好 Go 的 `math` 包两者都有，只是有一个小坑我们要弄明白；如果我们看 [`math.Cos`](https://golang.org/pkg/math/#Cos) 的描述：

> Cos returns the cosine of the radian argument x.

它要求角度是弧度。那什么是弧度？我们不再把一整圈的转动定义为 360 度，而是把一整圈定义为 2π 弧度。这样做是有充分理由的，但我们就不深入讨论了。

既然我们已经做了一些阅读、学习和思考，可以来写下一个测试了。

### 先写测试

这些数学既难又让人困惑。我没把握自己理解了发生的事情——所以让我们写个测试！我们不需要一次性解决整个问题——让我们先从算出某个特定时间秒针的正确角度（以弧度计）开始。

我会把之前在写的验收测试 _注释掉_，因为我在让这个测试通过的过程中不想被它分散注意力。

### 关于包的回顾

目前，我们的验收测试在 `clockface_test` 包中。我们的测试可以在 `clockface` 包之外——只要它们的名字以 `_test.go` 结尾，就能被运行。

我打算在 `clockface` 包 _内部_ 写这些弧度测试；它们可能永远不会被导出，并且一旦我对发生的事情有了更好的把握，它们可能会被删除（或移动）。我会把验收测试文件重命名为 `clockface_acceptance_test.go`，以便我可以创建一个 _新_ 文件 `clockface_test` 来测试秒数转弧度。

```go
package clockface

import (
	"math"
	"testing"
	"time"
)

func TestSecondsInRadians(t *testing.T) {
	thirtySeconds := time.Date(312, time.October, 28, 0, 0, 30, 0, time.UTC)
	want := math.Pi
	got := secondsInRadians(thirtySeconds)

	if want != got {
		t.Fatalf("Wanted %v radians, but got %v", want, got)
	}
}
```

这里我们测试的是过了 30 秒应该把秒针放在表盘的半圈处。这是我们第一次使用 `math` 包！如果一整圈是 2π 弧度，我们知道半圈应该正好是 π 弧度。`math.Pi` 给我们提供了 π 的值。

### 尝试运行测试

```
./clockface_test.go:12:9: undefined: secondsInRadians
```

### 写最少量的代码让测试运行起来，并检查失败的测试输出

```go
func secondsInRadians(t time.Time) float64 {
	return 0
}
```

```
clockface_test.go:15: Wanted 3.141592653589793 radians, but got 0
```

### 写足够的代码让它通过

```go
func secondsInRadians(t time.Time) float64 {
	return math.Pi
}
```

```
PASS
ok  	clockface	0.011s
```

### 重构

还没什么需要重构的

### 为新需求重复以上流程

现在我们可以扩展测试，覆盖更多场景。我会跳过一些步骤，直接展示一些已经重构过的测试代码——应该能很清楚地看出我是怎么走到这里的。

```go
func TestSecondsInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(0, 0, 30), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(0, 0, 45), (math.Pi / 2) * 3},
		{simpleTime(0, 0, 7), (math.Pi / 30) * 7},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondsInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

我加了几个辅助函数让写这个表驱动测试不那么繁琐。`testName` 把 time 转成数字手表格式 (HH:MM:SS)，`simpleTime` 只用我们真正关心的部分（同样是小时、分钟、秒）来构造一个 `time.Time`。它们在这里：

```go
func simpleTime(hours, minutes, seconds int) time.Time {
	return time.Date(312, time.October, 28, hours, minutes, seconds, 0, time.UTC)
}

func testName(t time.Time) string {
	return t.Format("15:04:05")
}
```

这两个函数应该能让这些测试（以及未来的测试）更容易写和维护。

这给了我们一些不错的测试输出：

```
clockface_test.go:24: Wanted 0 radians, but got 3.141592653589793

clockface_test.go:24: Wanted 4.71238898038469 radians, but got 3.141592653589793
```

是时候实现我们上面讨论过的所有数学了：

```go
func secondsInRadians(t time.Time) float64 {
	return float64(t.Second()) * (math.Pi / 30)
}
```

一秒是 (2π / 60) 弧度……约掉 2 我们得到 π/30 弧度。把它乘以秒数（作为 `float64`），现在所有测试应该都能通过了……

```
clockface_test.go:24: Wanted 3.141592653589793 radians, but got 3.1415926535897936
```

等等，什么？

### 浮点数太可怕了

浮点运算[出了名地不精确](https://0.30000000000000004.com/)。计算机其实只能很好地处理整数，对有理数则只能在某种程度上处理。十进制数会开始变得不精确，特别是当我们像在 `secondsInRadians` 函数里那样把它们除来乘去时。把 `math.Pi` 除以 30 然后再乘以 30，我们最终得到的是 _一个不再和 `math.Pi` 相同的数_。

有两种方法绕开这个问题：

1. 接受它
2. 通过重构方程来重构我们的函数

(1) 看起来似乎不太吸引人，但通常这是让浮点相等比较能工作的唯一办法。在某个无穷小的分数上不精确，对画一个表盘来说坦白讲并不重要，所以我们可以写一个函数，对我们的角度定义"足够接近"的相等。但有一个简单的办法可以恢复精度：我们重新整理方程，让我们不再先除后乘。我们可以全部用除法搞定。

所以与其

```
numberOfSeconds * π / 30
```

我们可以写

```
π / (30 / numberOfSeconds)
```

这是等价的。

在 Go 中：

```go
func secondsInRadians(t time.Time) float64 {
	return (math.Pi / (30 / (float64(t.Second()))))
}
```

我们就通过了。

```
PASS
ok      clockface     0.005s
```

它应该看起来[像这样](https://github.com/quii/learn-go-with-tests/tree/main/math/v3/clockface)。

### 关于除以零的注解

计算机通常不喜欢除以零，因为无穷大有点奇怪。

在 Go 中，如果你尝试显式除以零，你会得到一个编译错误。

```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println(10.0 / 0.0) // fails to compile
}
```

显然编译器不能总是预测出你会除以零，比如我们的 `t.Second()`

试试这个

```go
func main() {
	fmt.Println(10.0 / zero())
}

func zero() float64 {
	return 0.0
}
```

它会打印 `+Inf`（无穷大）。除以 +Inf 似乎结果是零，可以通过下面这段代码看到：

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Println(secondsinradians())
}

func zero() float64 {
	return 0.0
}

func secondsinradians() float64 {
	return (math.Pi / (30 / (float64(zero()))))
}
```

### 为新需求重复以上流程

所以我们已经覆盖了第一部分——我们知道秒针指向的角度（以弧度计）。现在我们需要算出坐标。

同样，让我们尽可能保持简单，只用 _单位圆_——半径为 1 的圆。这意味着我们所有的指针长度都会是 1，但好在数学会更容易消化。

### 先写测试

```go
func TestSecondHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 0, 30), Point{0, -1}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondHandPoint(c.time)
			if got != c.point {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

### 尝试运行测试

```
./clockface_test.go:40:11: undefined: secondHandPoint
```

### 写最少量的代码让测试运行起来，并检查失败的测试输出

```go
func secondHandPoint(t time.Time) Point {
	return Point{}
}
```

```
clockface_test.go:42: Wanted {0 -1} Point, but got {0 0}
```

### 写足够的代码让它通过

```go
func secondHandPoint(t time.Time) Point {
	return Point{0, -1}
}
```

```
PASS
ok  	clockface	0.007s
```

### 为新需求重复以上流程

```go
func TestSecondHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 0, 30), Point{0, -1}},
		{simpleTime(0, 0, 45), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondHandPoint(c.time)
			if got != c.point {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

### 尝试运行测试

```
clockface_test.go:43: Wanted {-1 0} Point, but got {0 -1}
```

### 写足够的代码让它通过

记得我们的单位圆图吗？

![picture of the unit circle with the x and y elements of a ray defined as cos(a) and sin(a) respectively, where a is the angle made by the ray with the x axis](<.gitbook/assets/unit_circle_params (1).png>)

还要记得我们想从 12 点钟方向（也就是 Y 轴）开始测量角度，而不是从 X 轴开始（那相当于测量秒针与 3 点钟方向的夹角）。

![unit circle ray defined from by angle from y axis](.gitbook/assets/unit_circle_12_oclock.png)

我们现在想要产生 X 和 Y 的方程。让我们把它写成秒数版本：

```go
func secondHandPoint(t time.Time) Point {
	angle := secondsInRadians(t)
	x := math.Sin(angle)
	y := math.Cos(angle)

	return Point{x, y}
}
```

现在我们得到

```
clockface_test.go:43: Wanted {0 -1} Point, but got {1.2246467991473515e-16 -1}

clockface_test.go:43: Wanted {-1 0} Point, but got {-1 -1.8369701987210272e-16}
```

等等，又是什么情况？看起来我们再次被浮点数诅咒了——这两个意外出现的数字都是 _无穷小的_——一直到第 16 位小数。所以我们再一次可以选择尝试提高精度，或者就说它们大致相等然后继续生活。

提高这些角度精度的一个选项是使用 `math/big` 包中的有理数类型 `Rat`。但既然目标是画一个 SVG 而不是登月，我想我们可以接受一点模糊。

```go
func TestSecondHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 0, 30), Point{0, -1}},
		{simpleTime(0, 0, 45), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}

func roughlyEqualFloat64(a, b float64) bool {
	const equalityThreshold = 1e-7
	return math.Abs(a-b) < equalityThreshold
}

func roughlyEqualPoint(a, b Point) bool {
	return roughlyEqualFloat64(a.X, b.X) &&
		roughlyEqualFloat64(a.Y, b.Y)
}
```

我们定义了两个函数来定义两个 `Point` 之间的近似相等——只要它们的 X 和 Y 元素相差在 0.0000001 之内就算相等。这仍然相当精确。

现在我们得到：

```
PASS
ok  	clockface	0.007s
```

### 重构

我对现状还挺满意的。

[现在它看起来是这样](https://github.com/quii/learn-go-with-tests/tree/main/math/v4/clockface)

### 为新需求重复以上流程

嗯，说 _新_ 不完全准确——其实我们现在能做的是让那个验收测试通过！让我们提醒自己它长什么样：

```go
func TestSecondHandAt30Seconds(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 30, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 150 + 90}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

### 尝试运行测试

```
clockface_acceptance_test.go:28: Got {150 60}, wanted {150 240}
```

### 写足够的代码让它通过

我们需要做三件事来把单位向量转换成 SVG 上的一个点：

1. 把它放大到指针的长度
2. 沿 X 轴翻转，以适配 SVG 原点在左上角
3. 平移到正确的位置（让它从原点 (150,150) 开始）

有趣的时间到了！

```go
// SecondHand is the unit vector of the second hand of an analogue clock at time `t`
// represented as a Point.
func SecondHand(t time.Time) Point {
	p := secondHandPoint(t)
	p = Point{p.X * 90, p.Y * 90}   // scale
	p = Point{p.X, -p.Y}            // flip
	p = Point{p.X + 150, p.Y + 150} // translate
	return p
}
```

按这个顺序进行缩放、翻转和平移。数学万岁！

```
PASS
ok  	clockface	0.007s
```

### 重构

这里有几个魔法数字应该被提取为常量，让我们这么做：

```go
const secondHandLength = 90
const clockCentreX = 150
const clockCentreY = 150

// SecondHand is the unit vector of the second hand of an analogue clock at time `t`
// represented as a Point.
func SecondHand(t time.Time) Point {
	p := secondHandPoint(t)
	p = Point{p.X * secondHandLength, p.Y * secondHandLength}
	p = Point{p.X, -p.Y}
	p = Point{p.X + clockCentreX, p.Y + clockCentreY} //translate
	return p
}
```

## 画时钟

嗯……至少先画秒针……

让我们做这件事——因为没有什么比明明价值就在那里等着输出到世界里去惊艳别人，却没能交付更糟糕的事了。我们来画一根秒针！

我们要在主 `clockface` 包目录下加一个新目录，名字（容易混淆地）也叫 `clockface`。在那里我们会放一个 `main` 包，用来生成创建 SVG 的二进制文件：

```
|-- clockface
|       |-- main.go
|-- clockface.go
|-- clockface_acceptance_test.go
|-- clockface_test.go
```

在 `main.go` 里，你会从这段代码开始，但要把 clockface 包的导入改成你自己的版本：

```go
package main

import (
	"fmt"
	"io"
	"os"
	"time"

	"learn-go-with-tests/math/clockface" // REPLACE THIS!
)

func main() {
	t := time.Now()
	sh := clockface.SecondHand(t)
	io.WriteString(os.Stdout, svgStart)
	io.WriteString(os.Stdout, bezel)
	io.WriteString(os.Stdout, secondHandTag(sh))
	io.WriteString(os.Stdout, svgEnd)
}

func secondHandTag(p clockface.Point) string {
	return fmt.Sprintf(`<line x1="150" y1="150" x2="%f" y2="%f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
}

const svgStart = `<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg"
     width="100%"
     height="100%"
     viewBox="0 0 300 300"
     version="2.0">`

const bezel = `<circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/>`

const svgEnd = `</svg>`
```

我可不会想凭 _这一坨_ 烂代码赢什么漂亮代码大奖——但它干活儿了。它把一个 SVG 写到了 `os.Stdout`——一次写一个字符串。

如果我们构建它

```
go build
```

并运行它，把输出送到一个文件里

```
./clockface > clock.svg
```

我们应该能看到类似这样的东西

![a clock with only a second hand](.gitbook/assets/clock.svg)

[代码看起来是这样](https://github.com/quii/learn-go-with-tests/tree/main/math/v6/clockface)。

### 重构

这有点臭。嗯，倒不是 _特别_ 臭，但我对它不满意。

1. 整个 `SecondHand` 函数 _极度_ 绑定于 SVG……虽然它没提到 SVG 也没真正生产 SVG……
2. ……同时我也没在测试任何 SVG 代码。

是的，我猜我搞砸了。这感觉不对。让我们试着用一个更以 SVG 为中心的测试来挽回。

我们有什么选择？嗯，我们可以试着测试 `SVGWriter` 喷出的字符里包含我们对某个特定时间所期望的那种 SVG 标签。例如：

```go
func TestSVGWriterAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	var b strings.Builder
	clockface.SVGWriter(&b, tm)
	got := b.String()

	want := `<line x1="150" y1="150" x2="150" y2="60"`

	if !strings.Contains(got, want) {
		t.Errorf("Expected to find the second hand %v, in the SVG output %v", want, got)
	}
}
```

但这真的算改进吗？

它不仅在我没有产生有效 SVG 的情况下也会通过（因为它只测试某个字符串是否出现在输出里），而且如果我对那个字符串做最小、不重要的改动（比如在属性间多加一个空格）它也会失败。

_最大_ 的味道是我在测试一个数据结构——XML——通过看它作为一系列字符的表示——作为一个字符串。这 _永远、永远_ 不是个好主意，因为它会产生我上面提到的那些问题：一个既太脆弱又不够敏感的测试。一个测错东西的测试！

所以唯一的解决方案是把输出 _作为 XML_ 来测试。要做到这一点我们需要解析它。

## 解析 XML

[`encoding/xml`](https://pkg.go.dev/encoding/xml) 是 Go 中处理简单 XML 解析所有事项的包。

函数 [`xml.Unmarshal`](https://pkg.go.dev/encoding/xml#Unmarshal) 接收一个 `[]byte` 的 XML 数据，以及一个指向某个结构体的指针来反序列化进去。

所以我们需要一个结构体把我们的 XML 反序列化进去。我们可以花一些时间研究所有节点和属性的正确名字以及如何写出正确的结构，但幸运的是有人写了 [`zek`](https://github.com/miku/zek)，一个能为我们自动完成所有这些艰苦工作的程序。更棒的是，还有一个在线版本在 [https://xml-to-go.github.io/](https://xml-to-go.github.io/)。只需把文件顶部的 SVG 粘贴到一个框里——砰——就出来了：

```go
type Svg struct {
	XMLName xml.Name `xml:"svg"`
	Text    string   `xml:",chardata"`
	Xmlns   string   `xml:"xmlns,attr"`
	Width   string   `xml:"width,attr"`
	Height  string   `xml:"height,attr"`
	ViewBox string   `xml:"viewBox,attr"`
	Version string   `xml:"version,attr"`
	Circle  struct {
		Text  string `xml:",chardata"`
		Cx    string `xml:"cx,attr"`
		Cy    string `xml:"cy,attr"`
		R     string `xml:"r,attr"`
		Style string `xml:"style,attr"`
	} `xml:"circle"`
	Line []struct {
		Text  string `xml:",chardata"`
		X1    string `xml:"x1,attr"`
		Y1    string `xml:"y1,attr"`
		X2    string `xml:"x2,attr"`
		Y2    string `xml:"y2,attr"`
		Style string `xml:"style,attr"`
	} `xml:"line"`
}
```

如果需要我们可以对它做一些调整（比如把结构体的名字改成 `SVG`），但作为开始它绝对足够好了。把这个结构体粘贴到 `clockface_acceptance_test` 文件里，让我们用它写一个测试：

```go
func TestSVGWriterAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	b := bytes.Buffer{}
	clockface.SVGWriter(&b, tm)

	svg := Svg{}
	xml.Unmarshal(b.Bytes(), &svg)

	x2 := "150"
	y2 := "60"

	for _, line := range svg.Line {
		if line.X2 == x2 && line.Y2 == y2 {
			return
		}
	}

	t.Errorf("Expected to find the second hand with x2 of %+v and y2 of %+v, in the SVG output %v", x2, y2, b.String())
}
```

我们把 `clockface.SVGWriter` 的输出写到一个 `bytes.Buffer`，然后 `Unmarshal` 进一个 `Svg`。然后我们查看 `Svg` 中的每个 `Line`，看是否有任何一个有期望的 `X2` 和 `Y2` 值。如果匹配上了我们提前返回（让测试通过）；否则我们用一条（希望能）提供信息的消息让它失败。

```sh
./clockface_acceptance_test.go:41:2: undefined: clockface.SVGWriter
```

看起来我们最好创建 `SVGWriter.go`……

```go
package clockface

import (
	"fmt"
	"io"
	"time"
)

const (
	secondHandLength = 90
	clockCentreX     = 150
	clockCentreY     = 150
)

// SVGWriter writes an SVG representation of an analogue clock, showing the time t, to the writer w
func SVGWriter(w io.Writer, t time.Time) {
	io.WriteString(w, svgStart)
	io.WriteString(w, bezel)
	secondHand(w, t)
	io.WriteString(w, svgEnd)
}

func secondHand(w io.Writer, t time.Time) {
	p := secondHandPoint(t)
	p = Point{p.X * secondHandLength, p.Y * secondHandLength} // scale
	p = Point{p.X, -p.Y}                                      // flip
	p = Point{p.X + clockCentreX, p.Y + clockCentreY}         // translate
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%f" y2="%f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
}

const svgStart = `<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg"
     width="100%"
     height="100%"
     viewBox="0 0 300 300"
     version="2.0">`

const bezel = `<circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/>`

const svgEnd = `</svg>`
```

最美的 SVG writer？不是。但希望它能干活儿……

```
clockface_acceptance_test.go:56: Expected to find the second hand with x2 of 150 and y2 of 60, in the SVG output <?xml version="1.0" encoding="UTF-8" standalone="no"?>
    <!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
    <svg xmlns="http://www.w3.org/2000/svg"
         width="100%"
         height="100%"
         viewBox="0 0 300 300"
         version="2.0"><circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/><line x1="150" y1="150" x2="150.000000" y2="60.000000" style="fill:none;stroke:#f00;stroke-width:3px;"/></svg>
```

哎哟！`%f` 格式指令把我们的坐标按默认精度——六位小数——打印出来。我们应该明确表示我们对坐标期望的精度。我们就说三位小数。

```go
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
```

然后在我们更新测试中的预期之后

```go
	x2 := "150.000"
	y2 := "60.000"
```

我们得到：

```
PASS
ok  	clockface	0.006s
```

我们现在可以缩短 `main` 函数：

```go
package main

import (
	"os"
	"time"

	"learn-go-with-tests/math/clockface"
)

func main() {
	t := time.Now()
	clockface.SVGWriter(os.Stdout, t)
}
```

[现在的样子](https://github.com/quii/learn-go-with-tests/tree/main/math/v7b/clockface)就是这样。

我们可以遵循同样的模式给另一个时间写一个测试，但在那之前……

### 重构

有三件事很突出：

1. 我们没有真正测试我们需要确保存在的所有信息——比如 `x1` 的值呢？
2. 而且，那些 `x1` 等属性其实不真的是 `string`，对吧？它们是数字！
3. 我真的关心指针的 `style` 吗？或者说，关心 `zak` 生成的那个空 `Text` 节点吗？

我们可以做得更好。让我们对 `Svg` 结构体和测试做一些调整，把一切都收紧。

```go
type SVG struct {
	XMLName xml.Name `xml:"svg"`
	Xmlns   string   `xml:"xmlns,attr"`
	Width   string   `xml:"width,attr"`
	Height  string   `xml:"height,attr"`
	ViewBox string   `xml:"viewBox,attr"`
	Version string   `xml:"version,attr"`
	Circle  Circle   `xml:"circle"`
	Line    []Line   `xml:"line"`
}

type Circle struct {
	Cx float64 `xml:"cx,attr"`
	Cy float64 `xml:"cy,attr"`
	R  float64 `xml:"r,attr"`
}

type Line struct {
	X1 float64 `xml:"x1,attr"`
	Y1 float64 `xml:"y1,attr"`
	X2 float64 `xml:"x2,attr"`
	Y2 float64 `xml:"y2,attr"`
}
```

这里我

* 把结构体中重要的部分变成命名类型——`Line` 和 `Circle`
* 把数字属性从 `string` 改成了 `float64`
* 删掉了未使用的属性，如 `Style` 和 `Text`
* 把 `Svg` 重命名成了 `SVG`，因为 _这是该做的事_。

这能让我们对要找的那条线做更精确的断言：

```go
func TestSVGWriterAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)
	b := bytes.Buffer{}

	clockface.SVGWriter(&b, tm)

	svg := SVG{}

	xml.Unmarshal(b.Bytes(), &svg)

	want := Line{150, 150, 150, 60}

	for _, line := range svg.Line {
		if line == want {
			return
		}
	}

	t.Errorf("Expected to find the second hand line %+v, in the SVG lines %+v", want, svg.Line)
}
```

最后，我们可以借鉴单元测试的表驱动思路，写一个辅助函数 `containsLine(line Line, lines []Line) bool` 来真正让这些测试发光：

```go
func TestSVGWriterSecondHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(0, 0, 0),
			Line{150, 150, 150, 60},
		},
		{
			simpleTime(0, 0, 30),
			Line{150, 150, 150, 240},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the second hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}

func containsLine(l Line, ls []Line) bool {
	for _, line := range ls {
		if line == l {
			return true
		}
	}
	return false
}
```

[它现在的样子](https://github.com/quii/learn-go-with-tests/tree/main/math/v7c/clockface)就在这里

现在 _这_ 才是我说的验收测试！

### 先写测试

至此秒针搞定。现在让我们开始做分针。

```go
func TestSVGWriterMinuteHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(0, 0, 0),
			Line{150, 150, 150, 70},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the minute hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}
```

### 尝试运行测试

```
clockface_acceptance_test.go:87: Expected to find the minute hand line {X1:150 Y1:150 X2:150 Y2:70}, in the SVG lines [{X1:150 Y1:150 X2:150 Y2:60}]
```

我们最好开始构建其他的时钟指针。和秒针的测试一样，我们可以迭代地产出下面这套测试。同样在我们让它工作期间我们会注释掉验收测试：

```go
func TestMinutesInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(0, 30, 0), math.Pi},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minutesInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### 尝试运行测试

```
./clockface_test.go:59:11: undefined: minutesInRadians
```

### 写最少量的代码让测试运行起来，并检查失败的测试输出

```go
func minutesInRadians(t time.Time) float64 {
	return math.Pi
}
```

### 为新需求重复以上流程

好吧——现在让我们做一些 _真正的_ 工作。我们可以把分针建模成只在每整分钟移动——所以它会从 30 分钟 "跳" 到 31 分钟，中间不动。但这看起来会有点垃圾。我们想要的是它每秒钟移动 _一点点_。

```go
func TestMinutesInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(0, 30, 0), math.Pi},
		{simpleTime(0, 0, 7), 7 * (math.Pi / (30 * 60))},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minutesInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

那一点点是多少？嗯……

* 一分钟有六十秒
* 半圈圆有三十分钟（`math.Pi` 弧度）
* 所以半圈是 `30 * 60` 秒。
* 所以如果时间是过了整点 7 秒……
* ……我们期望分针在 12 点钟过 `7 * (math.Pi / (30 * 60))` 弧度的位置。

### 尝试运行测试

```
clockface_test.go:62: Wanted 0.012217304763960306 radians, but got 3.141592653589793
```

### 写足够的代码让它通过

用 Jennifer Aniston 永恒的那句台词：[科学环节来了](https://www.youtube.com/watch?v=29Im23SPNok)

```go
func minutesInRadians(t time.Time) float64 {
	return (secondsInRadians(t) / 60) +
		(math.Pi / (30 / float64(t.Minute())))
}
```

与其每秒都从头计算分针在表盘上要移动多远，我们可以利用 `secondsInRadians` 函数。每过一秒分针移动的角度是秒针移动角度的 1/60。

```go
secondsInRadians(t) / 60
```

然后我们再加上分钟的移动量——和秒针的移动方式类似。

```go
math.Pi / (30 / float64(t.Minute()))
```

然后……

```
PASS
ok  	clockface	0.007s
```

简单又轻松。[现在的样子](https://github.com/quii/learn-go-with-tests/tree/main/math/v8/clockface/clockface_acceptance_test.go)是这样。

### 为新需求重复以上流程

我应该给 `minutesInRadians` 测试再加一些用例吗？目前只有两个。在我转去测试 `minuteHandPoint` 函数之前需要多少用例？

我最喜欢的 TDD 名言之一，常被归功于 Kent Beck：

> Write tests until fear is transformed into boredom.

而坦白讲，我已经厌倦了测试这个函数。我有信心知道它怎么工作。所以是时候转向下一个了。

### 先写测试

```go
func TestMinuteHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 30, 0), Point{0, -1}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minuteHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

### 尝试运行测试

```
./clockface_test.go:79:11: undefined: minuteHandPoint
```

### 写最少量的代码让测试运行起来，并检查失败的测试输出

```go
func minuteHandPoint(t time.Time) Point {
	return Point{}
}
```

```
clockface_test.go:80: Wanted {0 -1} Point, but got {0 0}
```

### 写足够的代码让它通过

```go
func minuteHandPoint(t time.Time) Point {
	return Point{0, -1}
}
```

```
PASS
ok  	clockface	0.007s
```

### 为新需求重复以上流程

现在做点真正的活儿

```go
func TestMinuteHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 30, 0), Point{0, -1}},
		{simpleTime(0, 45, 0), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minuteHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

```
clockface_test.go:81: Wanted {-1 0} Point, but got {0 -1}
```

### 写足够的代码让它通过

把 `secondHandPoint` 函数复制粘贴一下，做一些小改动应该就行……

```go
func minuteHandPoint(t time.Time) Point {
	angle := minutesInRadians(t)
	x := math.Sin(angle)
	y := math.Cos(angle)

	return Point{x, y}
}
```

```
PASS
ok  	clockface	0.009s
```

### 重构

`minuteHandPoint` 和 `secondHandPoint` 之间确实有些重复——我知道是因为我们刚刚是复制粘贴一个来做的另一个。让我们用一个函数把它们 DRY 掉。

```go
func angleToPoint(angle float64) Point {
	x := math.Sin(angle)
	y := math.Cos(angle)

	return Point{x, y}
}
```

然后我们可以把 `minuteHandPoint` 和 `secondHandPoint` 改写成一行：

```go
func minuteHandPoint(t time.Time) Point {
	return angleToPoint(minutesInRadians(t))
}
```

```go
func secondHandPoint(t time.Time) Point {
	return angleToPoint(secondsInRadians(t))
}
```

```
PASS
ok  	clockface	0.007s
```

现在我们可以取消验收测试的注释，开始画分针。

### 写足够的代码让它通过

`minuteHand` 函数是 `secondHand` 的复制粘贴，做了一些小调整，比如声明一个 `minuteHandLength`：

```go
const minuteHandLength = 80

//...

func minuteHand(w io.Writer, t time.Time) {
	p := minuteHandPoint(t)
	p = Point{p.X * minuteHandLength, p.Y * minuteHandLength}
	p = Point{p.X, -p.Y}
	p = Point{p.X + clockCentreX, p.Y + clockCentreY}
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#000;stroke-width:3px;"/>`, p.X, p.Y)
}
```

并在 `SVGWriter` 函数里调用它：

```go
func SVGWriter(w io.Writer, t time.Time) {
	io.WriteString(w, svgStart)
	io.WriteString(w, bezel)
	secondHand(w, t)
	minuteHand(w, t)
	io.WriteString(w, svgEnd)
}
```

现在我们应该看到 `TestSVGWriterMinuteHand` 通过：

```
PASS
ok  	clockface	0.006s
```

但布丁好不好吃，得吃了才知道——如果我们现在编译并运行 `clockface` 程序，应该能看到类似这样的东西

![a clock with second and minute hands](<.gitbook/assets/clock (1).svg>)

### 重构

让我们去掉 `secondHand` 和 `minuteHand` 函数中的重复，把所有缩放、翻转和平移的逻辑放到一个地方。

```go
func secondHand(w io.Writer, t time.Time) {
	p := makeHand(secondHandPoint(t), secondHandLength)
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
}

func minuteHand(w io.Writer, t time.Time) {
	p := makeHand(minuteHandPoint(t), minuteHandLength)
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#000;stroke-width:3px;"/>`, p.X, p.Y)
}

func makeHand(p Point, length float64) Point {
	p = Point{p.X * length, p.Y * length}
	p = Point{p.X, -p.Y}
	return Point{p.X + clockCentreX, p.Y + clockCentreY}
}
```

```
PASS
ok  	clockface	0.007s
```

[到这里我们的进度](https://github.com/quii/learn-go-with-tests/tree/main/math/v9/clockface)。

到这……就只剩时针了！

### 先写测试

```go
func TestSVGWriterHourHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(6, 0, 0),
			Line{150, 150, 150, 200},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the hour hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}
```

### 尝试运行测试

```
clockface_acceptance_test.go:113: Expected to find the hour hand line {X1:150 Y1:150 X2:150 Y2:200}, in the SVG lines [{X1:150 Y1:150 X2:150 Y2:60} {X1:150 Y1:150 X2:150 Y2:70}]
```

同样地，让我们把这个先注释掉，等我们用更底层的测试覆盖之后再说：

### 先写测试

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### 尝试运行测试

```
./clockface_test.go:97:11: undefined: hoursInRadians
```

### 写最少量的代码让测试运行起来，并检查失败的测试输出

```go
func hoursInRadians(t time.Time) float64 {
	return math.Pi
}
```

```
PASS
ok  	clockface	0.007s
```

### 为新需求重复以上流程

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### 尝试运行测试

```
clockface_test.go:100: Wanted 0 radians, but got 3.141592653589793
```

### 写足够的代码让它通过

```go
func hoursInRadians(t time.Time) float64 {
	return (math.Pi / (6 / float64(t.Hour())))
}
```

### 为新需求重复以上流程

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(21, 0, 0), math.Pi * 1.5},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### 尝试运行测试

```
clockface_test.go:101: Wanted 4.71238898038469 radians, but got 10.995574287564276
```

### 写足够的代码让它通过

```go
func hoursInRadians(t time.Time) float64 {
	return (math.Pi / (6 / (float64(t.Hour() % 12))))
}
```

记住，这不是 24 小时制时钟；我们必须用取模运算符来获取当前小时除以 12 的余数。

```
PASS
ok  	learn-go-with-tests/math/clockface	0.008s
```

### 先写测试

现在让我们试着根据已经过去的分钟和秒来移动时针。

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(21, 0, 0), math.Pi * 1.5},
		{simpleTime(0, 1, 30), math.Pi / ((6 * 60 * 60) / 90)},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### 尝试运行测试

```
clockface_test.go:102: Wanted 0.013089969389957472 radians, but got 0
```

### 写足够的代码让它通过

同样，需要思考一下。我们需要为分钟和秒钟都把时针稍微移动一点。幸运的是我们已经有一个分钟和秒钟的角度可用——`minutesInRadians` 返回的那个。我们可以复用它！

所以唯一的问题是要把这个角度的大小缩小多少倍。对分针来说一整圈是一小时，但对时针来说是十二小时。所以我们就把 `minutesInRadians` 返回的角度除以十二：

```go
func hoursInRadians(t time.Time) float64 {
	return (minutesInRadians(t) / 12) +
		(math.Pi / (6 / float64(t.Hour()%12)))
}
```

然后看：

```
clockface_test.go:104: Wanted 0.013089969389957472 radians, but got 0.01308996938995747
```

浮点运算又来了。

让我们更新测试，使用 `roughlyEqualFloat64` 来比较角度。

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(21, 0, 0), math.Pi * 1.5},
		{simpleTime(0, 1, 30), math.Pi / ((6 * 60 * 60) / 90)},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if !roughlyEqualFloat64(got, c.angle) {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

```
PASS
ok  	clockface	0.007s
```

### 重构

如果我们要在 _一个_ 弧度测试中使用 `roughlyEqualFloat64`，那大概应该 _所有_ 测试都用它。这是一个简单干净的重构，做完之后[看起来是这样](https://github.com/quii/learn-go-with-tests/tree/main/math/v10/clockface)。

## 时针的点

好了，是时候通过算出单位向量来计算时针点要去哪里了。

### 先写测试

```go
func TestHourHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(6, 0, 0), Point{0, -1}},
		{simpleTime(21, 0, 0), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hourHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

等等，我要 _一次_ 写 _两个_ 测试用例？这是不是 _糟糕的 TDD_？

### 关于 TDD 的狂热

测试驱动开发不是一种宗教。有些人可能表现得像是——通常是那些不做 TDD 但乐于在 Twitter 或 Dev.to 上抱怨它只是狂热分子才做的事，而他们不写测试是"务实"的。但它不是宗教。它是一个工具。

我 _知道_ 这两个测试会是什么——我以完全相同的方式测过另外两个时钟指针——而且我已经知道我的实现会是什么——我在分针迭代时已经写过一个把角度变成点的通用函数。

我不会为了 TDD 仪式而硬要走完那一套。TDD 是一种帮助我更好理解我正在写的代码——以及我即将写的代码——的技术。TDD 给我反馈、知识和洞察。但如果我已经有那些知识，就没有理由为了仪式而走流程。无论是测试还是 TDD 本身都不是目的。

我的信心增加了，所以我感觉可以迈更大的步子。我会"跳过"几个步骤，因为我知道我在哪、知道我要去哪、而且我以前走过这条路。

不过也要注意：我不是完全跳过写测试——我还是先写测试。它们只是以更不细的颗粒度出现而已。

### 尝试运行测试

```
./clockface_test.go:119:11: undefined: hourHandPoint
```

### 写足够的代码让它通过

```go
func hourHandPoint(t time.Time) Point {
	return angleToPoint(hoursInRadians(t))
}
```

如我所说，我知道我在哪里，知道我要去哪里。何必假装不是？如果我错了测试很快就会告诉我。

```
PASS
ok  	learn-go-with-tests/math/clockface	0.009s
```

## 画时针

最后我们来画时针。我们可以把那个验收测试取消注释加进来：

```go
func TestSVGWriterHourHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(6, 0, 0),
			Line{150, 150, 150, 200},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the hour hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}
```

### 尝试运行测试

```
clockface_acceptance_test.go:113: Expected to find the hour hand line {X1:150 Y1:150 X2:150 Y2:200},
    in the SVG lines [{X1:150 Y1:150 X2:150 Y2:60} {X1:150 Y1:150 X2:150 Y2:70}]
```

### 写足够的代码让它通过

我们现在可以对 SVG writing 的常量和函数做最后的调整：

```go
const (
	secondHandLength = 90
	minuteHandLength = 80
	hourHandLength   = 50
	clockCentreX     = 150
	clockCentreY     = 150
)

// SVGWriter writes an SVG representation of an analogue clock, showing the time t, to the writer w
func SVGWriter(w io.Writer, t time.Time) {
	io.WriteString(w, svgStart)
	io.WriteString(w, bezel)
	secondHand(w, t)
	minuteHand(w, t)
	hourHand(w, t)
	io.WriteString(w, svgEnd)
}

// ...

func hourHand(w io.Writer, t time.Time) {
	p := makeHand(hourHandPoint(t), hourHandLength)
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#000;stroke-width:3px;"/>`, p.X, p.Y)
}

```

然后……

```
ok  	clockface	0.007s
```

让我们编译并运行 `clockface` 程序检查一下。

![a clock](<.gitbook/assets/clock (2).svg>)

### 重构

看看 `clockface.go`，里面有一些"魔法数字"。它们都是基于一个表盘半圈有多少小时/分钟/秒。让我们重构来明确它们的含义。

```go
const (
	secondsInHalfClock = 30
	secondsInClock     = 2 * secondsInHalfClock
	minutesInHalfClock = 30
	minutesInClock     = 2 * minutesInHalfClock
	hoursInHalfClock   = 6
	hoursInClock       = 2 * hoursInHalfClock
)
```

为什么这么做？嗯，它让方程中每个数字 _的含义_ 变得明确。如果——_当_ 我们以后回到这段代码时——这些名字会帮助我们理解发生了什么。

而且，万一我们想做一些非常非常奇怪的时钟——比如时针有 4 小时、秒针 20 秒的——这些常量很容易就能变成参数。我们正在帮自己留着那扇门（即使我们永远不会走过去）。

## 总结

我们还需要做什么吗？

首先，让我们给自己一个鼓励——我们写出了一个能生成 SVG 表盘的程序。它能工作而且很棒。它只能做一种表盘——但那也没关系！或许你也只 _需要_ 一种表盘。一个程序解决一个具体问题、不做其他，没什么不对。

### 一个程序……和一个库

但我们写的代码 _确实_ 解决了一系列与画表盘相关的更通用的问题。因为我们用测试来思考问题的每个小部分（彼此独立），并且通过函数把那种独立性固化下来，我们已经构建了一个相当合理的小型 API 用于表盘计算。

我们可以在这个项目上继续工作，把它变成更通用的东西——一个用于计算表盘角度和/或向量的库。

事实上，把库和程序一起提供 _是个非常好的主意_。它对我们没什么成本，同时能增加程序的实用性，并帮助记录它的工作方式。

> APIs should come with programs, and vice versa. An API that you must write C code to use, which cannot be invoked easily from the command line, is harder to learn and use. And contrariwise, it's a royal pain to have interfaces whose only open, documented form is a program, so you cannot invoke them easily from a C program. -- Henry Spencer, in _The Art of Unix Programming_

在[我对这个程序的最终版本](https://github.com/quii/learn-go-with-tests/tree/main/math/vFinal/clockface)中，我把 `clockface` 中未导出的函数变成了库的公开 API，提供了为每根时钟指针计算角度和单位向量的函数。我也把 SVG 生成部分拆分到了它自己的包 `svg`，然后由 `clockface` 程序直接使用。当然，每个函数和包我都写了文档。

说到 SVG……

### 最有价值的测试

我相信你已经注意到处理 SVG 最复杂的代码并不在我们的应用代码里；它在测试代码里。这应该让我们感到不舒服吗？我们是不是该做点什么，比如

* 用 `text/template` 中的模板？
* 用一个 XML 库（就像我们在测试中做的）？
* 用一个 SVG 库？

我们可以把代码重构成做这些事情的任何一种，而且我们能这样做，是因为我们 _怎么_ 生产 SVG 并不重要，重要的是我们 _生产了什么_——_一个 SVG_。因此，我们系统中需要最了解 SVG 的部分——需要对什么构成 SVG 最严格的部分——是对 SVG 输出的测试：它需要有足够的关于 SVG 的上下文和知识，这样我们才能确信我们正在输出一个 SVG。SVG 的 _是什么_ 存活在我们的测试里；_怎么做_ 在代码里。

我们或许会觉得在 SVG 测试上倾注大量时间和精力很奇怪——引入一个 XML 库、解析 XML、重构结构体——但那段测试代码是我们代码库中有价值的一部分——可能比当前的生产代码更有价值。它将帮助保证输出始终是有效的 SVG，无论我们选择用什么来生成它。

测试不是二等公民——它们不是"用完即扔"的代码。好的测试会比它们正在测试的那个版本的代码活得久得多。你不应该觉得你"花太多时间"写测试。这是一项投资。

1. 简而言之它让用圆做微积分更容易，因为如果用普通度数，π 会作为角度不断出现，所以如果你用 π 来数你的角度，所有方程都会变得更简单。
