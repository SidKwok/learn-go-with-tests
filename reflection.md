# 反射

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/reflection)**

[来自 Twitter](https://twitter.com/peterbourgon/status/1011403901419937792?s=09)

> golang challenge: write a function `walk(x interface{}, fn func(string))` which takes a struct `x` and calls `fn` for all strings fields found inside. difficulty level: recursively.

要做到这一点，我们需要使用_反射_。

> Reflection in computing is the ability of a program to examine its own structure, particularly through types; it's a form of metaprogramming. It's also a great source of confusion.

来自 [The Go Blog: Reflection](https://blog.golang.org/laws-of-reflection)

## 什么是 `interface{}`？

我们一直享受着 Go 在那些处理已知类型（如 `string`、`int`，以及我们自己定义的类型如 `BankAccount`）的函数中提供的类型安全。

这意味着我们免费获得了一些文档，并且如果你试图给函数传错类型，编译器会抱怨。

但你可能会遇到一些场景，你想写一个函数，而它在编译期并不知道类型。

Go 通过 `interface{}` 类型让我们可以绕过这个问题，你可以把它理解为_任意_类型（事实上，在 Go 中 `any` 是 `interface{}` 的[别名](https://cs.opensource.google/go/go/+/master:src/builtin/builtin.go;drc=master;l=95)）。

所以 `walk(x interface{}, fn func(string))` 可以接受任何类型的 `x`。

### 那为什么不让所有东西都用 `interface{}`，写出真正灵活的函数？

- 作为接收 `interface{}` 的函数的使用者，你失去了类型安全。如果你本意是把 `string` 类型的 `Herd.species` 传入函数，结果不小心传了 `int` 类型的 `Herd.count`，编译器无法告诉你这个错误。你也完全不知道_什么_是允许传给函数的。比如知道一个函数接受一个 `UserService`，这就非常有用。
- 作为这种函数的编写者，你必须能够检视_任何_被传入的东西，并努力弄清它的类型以及你能用它做什么。这要通过_反射_来完成。这种做法相当笨拙、可读性差，并且通常性能也较低（因为你必须在运行时做检查）。

简而言之，只有真的需要时才用反射。

如果你想写多态的函数，可以考虑围绕一个接口（不是 `interface{}`，容易混淆）来设计，让用户在实现你函数所需的方法后，可以把多种类型用在你的函数上。

我们的函数将需要能处理大量不同的东西。一如既往，我们会采用迭代的方式，为每个想支持的新东西写测试，并在过程中不断重构，直到完成。

## 先写测试

我们会想用一个有 string 字段的 struct（`x`）来调用我们的函数。然后我们可以用 spy 监视传入的函数（`fn`）来看看它是否被调用。

```go
func TestWalk(t *testing.T) {

	expected := "Chris"
	var got []string

	x := struct {
		Name string
	}{expected}

	walk(x, func(input string) {
		got = append(got, input)
	})

	if len(got) != 1 {
		t.Errorf("wrong number of function calls, got %d want %d", len(got), 1)
	}
}
```

- 我们要保存一个字符串切片（`got`），存放被 `walk` 传给 `fn` 的字符串。在前面的章节里，我们经常为此专门做一个类型来 spy 函数/方法的调用，但这次我们可以直接传一个匿名函数作为 `fn`，让它闭包捕获 `got`。
- 我们用了一个带 `Name` string 字段的匿名 `struct`，走最简单的"happy path"。
- 最后用 `x` 和 spy 调用 `walk`，目前先只检查 `got` 的长度，等基础能跑通了再做更具体的断言。

## 试着运行测试

```
./reflection_test.go:21:2: undefined: walk
```

## 写最少的代码让测试运行，并检查失败的测试输出

我们需要定义 `walk`

```go
func walk(x interface{}, fn func(input string)) {

}
```

再次尝试运行测试

```
=== RUN   TestWalk
--- FAIL: TestWalk (0.00s)
    reflection_test.go:19: wrong number of function calls, got 0 want 1
FAIL
```

## 写足够的代码让它通过

我们可以用任何字符串调用 spy 让它通过。

```go
func walk(x interface{}, fn func(input string)) {
	fn("I still can't believe South Korea beat Germany 2-0 to put them last in their group")
}
```

测试现在应该通过了。下一步要做的是对 `fn` 被调用时传入的内容做更具体的断言。

## 先写测试

把下面的内容加到现有测试里，检查传给 `fn` 的字符串是否正确

```go
if got[0] != expected {
	t.Errorf("got %q, want %q", got[0], expected)
}
```

## 试着运行测试

```
=== RUN   TestWalk
--- FAIL: TestWalk (0.00s)
    reflection_test.go:23: got 'I still can't believe South Korea beat Germany 2-0 to put them last in their group', want 'Chris'
FAIL
```

## 写足够的代码让它通过

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)
	field := val.Field(0)
	fn(field.String())
}
```

这段代码_非常不安全、非常天真_，但记住：当我们处于"红灯"（测试失败）时，我们的目标是写出尽可能少的代码。然后再写更多测试来解决我们的顾虑。

我们需要用反射来看一下 `x` 并尝试查看它的属性。

[reflect 包](https://pkg.go.dev/reflect)有一个函数 `ValueOf`，它会返回给定变量的 `Value`。这有让我们检视一个值的方法，包括它的字段——我们在下一行就用上了。

然后我们对传入的值做了一些非常乐观的假设：

- 我们看了第一个也是唯一一个字段。然而，可能根本就没有字段，那会引发 panic。
- 然后我们调用了 `String()`，它返回该值底层的字符串。但如果字段不是字符串，就错了。

## 重构

我们的代码在简单情况下能通过，但我们知道代码有许多缺陷。

我们要写一系列测试，传入不同的值，检查 `fn` 被调用时收到的字符串数组。

我们应该把测试重构成表驱动测试，便于继续测试新的场景。

```go
func TestWalk(t *testing.T) {

	cases := []struct {
		Name          string
		Input         interface{}
		ExpectedCalls []string
	}{
		{
			"struct with one string field",
			struct {
				Name string
			}{"Chris"},
			[]string{"Chris"},
		},
	}

	for _, test := range cases {
		t.Run(test.Name, func(t *testing.T) {
			var got []string
			walk(test.Input, func(input string) {
				got = append(got, input)
			})

			if !reflect.DeepEqual(got, test.ExpectedCalls) {
				t.Errorf("got %v, want %v", got, test.ExpectedCalls)
			}
		})
	}
}
```

现在我们可以轻松添加一个场景，看看如果有多个 string 字段会发生什么。

## 先写测试

把下面这个场景加到 `cases` 里。

```
{
    "struct with two string fields",
    struct {
        Name string
        City string
    }{"Chris", "London"},
    []string{"Chris", "London"},
},
```

## 试着运行测试

```
=== RUN   TestWalk/struct_with_two_string_fields
    --- FAIL: TestWalk/struct_with_two_string_fields (0.00s)
        reflection_test.go:40: got [Chris], want [Chris London]
```

## 写足够的代码让它通过

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)
		fn(field.String())
	}
}
```

`val` 有一个方法 `NumField`，它返回值中字段的数量。这让我们能遍历字段并调用 `fn`，从而通过测试。

## 重构

这里看不到能改进代码的明显重构，所以我们继续。

`walk` 的下一个缺陷是它假定每个字段都是 `string`。让我们为这个场景写一个测试。

## 先写测试

加上下面这个用例

```
{
    "struct with non string field",
    struct {
        Name string
        Age  int
    }{"Chris", 33},
    []string{"Chris"},
},
```

## 试着运行测试

```
=== RUN   TestWalk/struct_with_non_string_field
    --- FAIL: TestWalk/struct_with_non_string_field (0.00s)
        reflection_test.go:46: got [Chris <int Value>], want [Chris]
```

## 写足够的代码让它通过

我们需要检查字段的类型是 `string`。

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		if field.Kind() == reflect.String {
			fn(field.String())
		}
	}
}
```

我们可以通过检查它的 [`Kind`](https://pkg.go.dev/reflect#Kind) 来做到。

## 重构

再次，看起来代码目前已经足够合理。

下一个场景是：如果它不是一个"扁平"的 `struct` 呢？换句话说，如果我们有一个带嵌套字段的 `struct`，会发生什么？

## 先写测试

我们一直在用匿名 struct 语法即时声明类型来做测试，所以可以继续这样写

```
{
    "nested fields",
    struct {
        Name string
        Profile struct {
            Age  int
            City string
        }
    }{"Chris", struct {
        Age  int
        City string
    }{33, "London"}},
    []string{"Chris", "London"},
},
```

但可以看到，当出现内嵌的匿名 struct 时，语法变得有点乱。[有一个提案让这种语法变得更舒服](https://github.com/golang/go/issues/12854)。

我们直接重构一下，为这个场景定义一个具名类型，并在测试里引用它。这样会有一点间接性——测试用到的部分代码在测试外部——但读者应该能从初始化中推断出 `struct` 的结构。

把下面的类型声明加到你的测试文件中

```go
type Person struct {
	Name    string
	Profile Profile
}

type Profile struct {
	Age  int
	City string
}
```

现在我们可以把它加到 cases 里，读起来比之前清晰多了

```
{
    "nested fields",
    Person{
        "Chris",
        Profile{33, "London"},
    },
    []string{"Chris", "London"},
},
```

## 试着运行测试

```
=== RUN   TestWalk/Nested_fields
    --- FAIL: TestWalk/nested_fields (0.00s)
        reflection_test.go:54: got [Chris], want [Chris London]
```

问题在于我们只在类型层级的第一层迭代字段。

## 写足够的代码让它通过

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		if field.Kind() == reflect.String {
			fn(field.String())
		}

		if field.Kind() == reflect.Struct {
			walk(field.Interface(), fn)
		}
	}
}
```

解决方案非常简单，我们再次检查它的 `Kind`，如果恰好是 `struct`，就在那个内部 `struct` 上再调用一次 `walk`。

## 重构

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

当你在同一个值上做多次比较时，_一般来说_ 重构成 `switch` 会提升可读性，并让代码更易扩展。

那如果传入的 struct 值是一个指针呢？

## 先写测试

加上这个用例

```
{
    "pointers to things",
    &Person{
        "Chris",
        Profile{33, "London"},
    },
    []string{"Chris", "London"},
},
```

## 试着运行测试

```
=== RUN   TestWalk/pointers_to_things
panic: reflect: call of reflect.Value.NumField on ptr Value [recovered]
    panic: reflect: call of reflect.Value.NumField on ptr Value
```

## 写足够的代码让它通过

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Pointer {
		val = val.Elem()
	}

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

你不能在指针 `Value` 上使用 `NumField`，我们需要先用 `Elem()` 提取出底层的值，才能这么做。

## 重构

让我们把"从给定 `interface{}` 中提取 `reflect.Value`"这个职责封装到一个函数里。

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}

func getValue(x interface{}) reflect.Value {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Pointer {
		val = val.Elem()
	}

	return val
}
```

这其实_增加_了代码量，但我觉得抽象层级是合适的。

- 拿到 `x` 的 `reflect.Value` 来检视它，我不关心怎么拿到的。
- 遍历字段，根据其类型做该做的事。

接下来，我们要支持切片。

## 先写测试

```
{
    "slices",
    []Profile {
        {33, "London"},
        {34, "Reykjavík"},
    },
    []string{"London", "Reykjavík"},
},
```

## 试着运行测试

```
=== RUN   TestWalk/slices
panic: reflect: call of reflect.Value.NumField on slice Value [recovered]
    panic: reflect: call of reflect.Value.NumField on slice Value
```

## 写最少的代码让测试运行，并检查失败的测试输出

这跟之前的指针场景类似，我们试图在 `reflect.Value` 上调用 `NumField`，但它没有这个方法，因为它不是 struct。

## 写足够的代码让它通过

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	if val.Kind() == reflect.Slice {
		for i := 0; i < val.Len(); i++ {
			walk(val.Index(i).Interface(), fn)
		}
		return
	}

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

## 重构

它能工作，但很难看。不过别担心，我们有测试支撑的可工作代码，可以随心所欲地折腾。

如果稍微抽象地思考，我们想对以下两种情况调用 `walk`

- struct 中的每个字段
- 切片中的每个_东西_

我们当前的代码就是这么做的，但表达得并不清晰。我们只是在开头检查它是不是切片（用 `return` 阻止后面的代码执行），如果不是，就假设它是 struct。

让我们重做代码，先检查类型，再做对应的工作。

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	switch val.Kind() {
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walk(val.Field(i).Interface(), fn)
		}
	case reflect.Slice:
		for i := 0; i < val.Len(); i++ {
			walk(val.Index(i).Interface(), fn)
		}
	case reflect.String:
		fn(val.String())
	}
}
```

看起来好多了！如果是 struct 或切片，我们遍历它的值，对每个调用 `walk`。否则，如果是 `reflect.String`，我们就调用 `fn`。

不过对我来说，感觉还可以更好。"遍历字段/值，然后调用 `walk`"这一操作存在重复，但概念上它们是一样的。

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice:
		numberOfValues = val.Len()
		getField = val.Index
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

如果 `value` 是 `reflect.String`，我们就照常调用 `fn`。

否则，`switch` 会根据类型提取出两样东西

- 有多少字段
- 怎样提取 `Value`（`Field` 还是 `Index`）

一旦确定了这些，我们就可以遍历 `numberOfValues`，用 `getField` 函数的结果调用 `walk`。

做完这一步之后，处理数组就轻而易举了。

## 先写测试

加到 cases 里

```
{
    "arrays",
    [2]Profile {
        {33, "London"},
        {34, "Reykjavík"},
    },
    []string{"London", "Reykjavík"},
},
```

## 试着运行测试

```
=== RUN   TestWalk/arrays
    --- FAIL: TestWalk/arrays (0.00s)
        reflection_test.go:78: got [], want [London Reykjavík]
```

## 写足够的代码让它通过

数组的处理方式和切片相同，所以只需用逗号把它加到那个 case 里

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice, reflect.Array:
		numberOfValues = val.Len()
		getField = val.Index
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

我们要支持的下一个类型是 `map`。

## 先写测试

```
{
    "maps",
    map[string]string{
        "Cow": "Moo",
        "Sheep": "Baa",
    },
    []string{"Moo", "Baa"},
},
```

## 试着运行测试

```
=== RUN   TestWalk/maps
    --- FAIL: TestWalk/maps (0.00s)
        reflection_test.go:86: got [], want [Moo Baa]
```

## 写足够的代码让它通过

再次，如果你抽象地想一下，可以看到 `map` 和 `struct` 非常相似，只是它的键在编译期是未知的。

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice, reflect.Array:
		numberOfValues = val.Len()
		getField = val.Index
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walk(val.MapIndex(key).Interface(), fn)
		}
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

但是，按照设计，你不能通过下标从 map 里取值，只能通过_键_来取，所以这破坏了我们的抽象，糟糕。

## 重构

你现在感觉如何？当时看起来像是一个不错的抽象，但现在代码感觉有点别扭。

_这没什么！_ 重构是一段旅程，我们有时会犯错。TDD 的一个重要意义就在于它给了我们尝试这些事情的自由。

通过测试支撑的小步前进，这绝不是不可逆转的状况。我们就把它恢复到重构之前的样子。

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	}
}
```

我们引入了 `walkValue`，把 `switch` 内部对 `walk` 的调用 DRY 化，使它们只需要从 `val` 中提取 `reflect.Value` 即可。

### 最后一个问题

记住，Go 中的 map 不保证顺序。所以你的测试有时会失败，因为我们断言 `fn` 的调用是按特定顺序进行的。

为了解决这个问题，我们需要把对 map 的断言挪到一个不关心顺序的新测试里。

```go
t.Run("with maps", func(t *testing.T) {
	aMap := map[string]string{
		"Cow":   "Moo",
		"Sheep": "Baa",
	}

	var got []string
	walk(aMap, func(input string) {
		got = append(got, input)
	})

	assertContains(t, got, "Moo")
	assertContains(t, got, "Baa")
})
```

下面是 `assertContains` 的定义

```go
func assertContains(t testing.TB, haystack []string, needle string) {
	t.Helper()
	contains := false
	for _, x := range haystack {
		if x == needle {
			contains = true
		}
	}
	if !contains {
		t.Errorf("expected %v to contain %q but it didn't", haystack, needle)
	}
}
```

由于我们已经把 maps 抽到了新测试中，我们还没看到失败信息长什么样。在这里故意把 `with maps` 测试搞坏一下，这样你就能看到错误信息，然后再修好让所有测试都通过。

我们要支持的下一个类型是 `chan`。

## 先写测试

```go
t.Run("with channels", func(t *testing.T) {
	aChannel := make(chan Profile)

	go func() {
		aChannel <- Profile{33, "Berlin"}
		aChannel <- Profile{34, "Katowice"}
		close(aChannel)
	}()

	var got []string
	want := []string{"Berlin", "Katowice"}

	walk(aChannel, func(input string) {
		got = append(got, input)
	})

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

## 试着运行测试

```
--- FAIL: TestWalk (0.00s)
    --- FAIL: TestWalk/with_channels (0.00s)
        reflection_test.go:115: got [], want [Berlin Katowice]
```

## 写足够的代码让它通过

我们可以用 Recv() 遍历 channel 中发送的所有值，直到它被关闭

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	case reflect.Chan:
		for {
			if v, ok := val.Recv(); ok {
				walkValue(v)
			} else {
				break
			}
		}
	}
}
```
我们要支持的下一个类型是 `func`。

## 先写测试

```go
t.Run("with function", func(t *testing.T) {
	aFunction := func() (Profile, Profile) {
		return Profile{33, "Berlin"}, Profile{34, "Katowice"}
	}

	var got []string
	want := []string{"Berlin", "Katowice"}

	walk(aFunction, func(input string) {
		got = append(got, input)
	})

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

## 试着运行测试

```
--- FAIL: TestWalk (0.00s)
    --- FAIL: TestWalk/with_function (0.00s)
        reflection_test.go:132: got [], want [Berlin Katowice]
```

## 写足够的代码让它通过

在这个场景里，非零参数的函数似乎没什么意义。但我们应当允许任意的返回值。

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	case reflect.Chan:
		for v, ok := val.Recv(); ok; v, ok = val.Recv() {
			walkValue(v)
		}
	case reflect.Func:
		valFnResult := val.Call(nil)
		for _, res := range valFnResult {
			walkValue(res)
		}
	}
}
```

## 总结

- 介绍了 `reflect` 包中的一些概念。
- 用递归来遍历任意的数据结构。
- 事后看来做了一次糟糕的重构，但没必要太沮丧。通过有测试支撑的迭代式工作，这没什么大不了。
- 这只覆盖了反射的一小部分。[Go 博客上有一篇绝妙的文章涵盖了更多细节](https://blog.golang.org/laws-of-reflection)。
- 既然你现在了解了反射，请尽你所能避免使用它。
