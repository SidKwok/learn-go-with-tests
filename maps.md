# Maps

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/maps)**

在[数组与切片](arrays-and-slices.md)一章中，你看到了如何按顺序存储值。现在我们来看看一种通过 `key` 存储元素并快速查找的方式。

map 让你可以以类似字典的方式存储元素。你可以把 `key` 想成单词，把 `value` 想成定义。还有什么比构建一个我们自己的字典更好的方式来学习 map 呢？

首先，假设字典里已经有一些单词及其定义，当我们搜索一个单词时，应该返回它的定义。

## 先写测试

在 `dictionary_test.go` 中：

```go
package main

import "testing"

func TestSearch(t *testing.T) {
	dictionary := map[string]string{"test": "this is just a test"}

	got := Search(dictionary, "test")
	want := "this is just a test"

	if got != want {
		t.Errorf("got %q want %q given, %q", got, want, "test")
	}
}
```

声明一个 map 在某种程度上和声明数组类似。区别是它以 `map` 关键字开头并且需要两个类型。第一个是 key 的类型，写在 `[]` 里。第二个是 value 的类型，紧跟在 `[]` 之后。

key 的类型是特殊的，它只能是可比较的类型，因为如果没法判断两个 key 是否相等，我们就没办法保证拿到的是正确的 value。可比较类型的详细说明见[语言规范](https://golang.org/ref/spec#Comparison_operators)。

而 value 的类型可以是任何你想要的类型，甚至可以是另一个 map。

这个测试中其他的内容你应该都熟悉。

## 尝试运行测试

运行 `go test`，编译器会报错 `./dictionary_test.go:8:9: undefined: Search`。

## 写最少量的代码让测试运行起来，并检查输出

在 `dictionary.go` 中：

```go
package main

func Search(dictionary map[string]string, word string) string {
	return ""
}
```

现在你的测试应该会失败，并给出一条 *清晰的错误信息*：

`dictionary_test.go:12: got '' want 'this is just a test' given, 'test'`。

## 写足够的代码让测试通过

```go
func Search(dictionary map[string]string, word string) string {
	return dictionary[word]
}
```

从 map 里取值的方式和从数组取值一样，`map[key]`。

## 重构

```go
func TestSearch(t *testing.T) {
	dictionary := map[string]string{"test": "this is just a test"}

	got := Search(dictionary, "test")
	want := "this is just a test"

	assertStrings(t, got, want)
}

func assertStrings(t testing.TB, got, want string) {
	t.Helper()

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

我决定创建一个 `assertStrings` 辅助函数，让实现更通用。

### 使用自定义类型

我们可以围绕 map 创建一个新类型，并把 `Search` 做成方法，从而改善字典的用法。

在 `dictionary_test.go` 中：

```go
func TestSearch(t *testing.T) {
	dictionary := Dictionary{"test": "this is just a test"}

	got := dictionary.Search("test")
	want := "this is just a test"

	assertStrings(t, got, want)
}
```

我们开始使用 `Dictionary` 类型，虽然还没定义它。然后在 `Dictionary` 实例上调用 `Search`。

我们没必要修改 `assertStrings`。

在 `dictionary.go` 中：

```go
type Dictionary map[string]string

func (d Dictionary) Search(word string) string {
	return d[word]
}
```

这里我们创建了一个 `Dictionary` 类型，它是对 `map` 的一层薄封装。有了自定义类型之后，我们就可以创建 `Search` 方法了。

## 先写测试

基础的搜索实现起来很简单，但如果传入字典里没有的单词会怎样？

实际上我们什么都不会得到。这虽然挺好，因为程序还能继续跑，但有更好的做法。函数可以反馈"这个词不在字典里"。这样用户就不会困惑：到底是单词不存在还是单词没有定义（这一点对字典来说也许没什么用，但在其他场景里这种区分可能很关键）。

```go
func TestSearch(t *testing.T) {
	dictionary := Dictionary{"test": "this is just a test"}

	t.Run("known word", func(t *testing.T) {
		got, _ := dictionary.Search("test")
		want := "this is just a test"

		assertStrings(t, got, want)
	})

	t.Run("unknown word", func(t *testing.T) {
		_, err := dictionary.Search("unknown")
		want := "could not find the word you were looking for"

		if err == nil {
			t.Fatal("expected to get an error.")
		}

		assertStrings(t, err.Error(), want)
	})
}
```

在 Go 中处理这种场景的方式是返回第二个参数，是一个 `Error` 类型。

注意，正如我们在[指针与错误](./pointers-and-errors.md)那一章里看到的，要断言错误信息，
我们先检查错误不为 `nil`，然后用 `.Error()` 方法拿到字符串再传给断言。

## 尝试运行测试

它编译不过：

```
./dictionary_test.go:18:10: assignment mismatch: 2 variables but 1 values
```

## 写最少量的代码让测试运行起来，并检查输出

```go
func (d Dictionary) Search(word string) (string, error) {
	return d[word], nil
}
```

现在你的测试会失败，并给出一条更清晰的错误信息：

`dictionary_test.go:22: expected to get an error.`

## 写足够的代码让测试通过

```go
func (d Dictionary) Search(word string) (string, error) {
	definition, ok := d[word]
	if !ok {
		return "", errors.New("could not find the word you were looking for")
	}

	return definition, nil
}
```

为了让测试通过，我们利用了 map 查找的一个有趣特性：它可以返回两个值。第二个值是一个布尔值，表示 key 是否成功被找到。

这个特性让我们能够区分"不存在的单词"与"存在但没有定义的单词"。

## 重构

```go
var ErrNotFound = errors.New("could not find the word you were looking for")

func (d Dictionary) Search(word string) (string, error) {
	definition, ok := d[word]
	if !ok {
		return "", ErrNotFound
	}

	return definition, nil
}
```

我们可以通过把"魔法"错误抽取成一个变量来去掉它。这也能让我们的测试更好。

```go
t.Run("unknown word", func(t *testing.T) {
	_, got := dictionary.Search("unknown")
	if got == nil {
		t.Fatal("expected to get an error.")
	}
	assertError(t, got, ErrNotFound)
})
```
```go
func assertError(t testing.TB, got, want error) {
	t.Helper()

	if got != want {
		t.Errorf("got error %q want %q", got, want)
	}
}
```

通过创建一个新的辅助函数，我们简化了测试，并开始使用 `ErrNotFound` 变量，这样以后我们改了错误文本也不会让测试失败。

## 先写测试

我们已经有了一种很好的搜索字典的方式，但还没办法往字典里添加新单词。

```go
func TestAdd(t *testing.T) {
	dictionary := Dictionary{}
	dictionary.Add("test", "this is just a test")

	want := "this is just a test"
	got, err := dictionary.Search("test")
	if err != nil {
		t.Fatal("should find added word:", err)
	}

	assertStrings(t, got, want)
}
```

在这个测试里，我们利用 `Search` 函数让字典验证更简单一点。

## 写最少量的代码让测试运行起来，并检查输出

在 `dictionary.go` 中：

```go
func (d Dictionary) Add(word, definition string) {
}
```

现在你的测试应该会失败：

```
dictionary_test.go:31: should find added word: could not find the word you were looking for
```

## 写足够的代码让测试通过

```go
func (d Dictionary) Add(word, definition string) {
	d[word] = definition
}
```

往 map 里添加也和数组类似，你只需要指定一个 key 并把它设为某个 value。

### 指针、拷贝以及其他

map 的一个有趣特性是你可以修改它而不需要传入它的地址（例如 `&myMap`）。

这可能让 map _感觉_ 像是"引用类型"，[但正如 Dave Cheney 解释的那样](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it)它们并不是。

> map 值是一个指向 runtime.hmap 结构体的指针。

所以当你把一个 map 传给函数/方法时，你确实是在拷贝它，但只是拷贝指针那部分，而不是底层包含数据的数据结构。

map 有一个坑：它可能是 `nil` 值。`nil` map 在读取时表现得像空 map，但尝试往 `nil` map 里写入会触发运行时 panic。你可以在[这里](https://blog.golang.org/go-maps-in-action)读到更多关于 map 的内容。

因此，你不应该这样初始化一个 nil map 变量：

```go
var m map[string]string
```

相反，你可以初始化一个空 map 或者用 `make` 关键字来创建：

```go
var dictionary = map[string]string{}

// OR

var dictionary = make(map[string]string)
```

这两种方式都会创建一个空的 `hash map`，并把 `dictionary` 指向它。这样能确保你永远不会触发运行时 panic。

## 重构

我们的实现没什么可以重构的，但测试可以稍微简化一下。

```go
func TestAdd(t *testing.T) {
	dictionary := Dictionary{}
	word := "test"
	definition := "this is just a test"

	dictionary.Add(word, definition)

	assertDefinition(t, dictionary, word, definition)
}

func assertDefinition(t testing.TB, dictionary Dictionary, word, definition string) {
	t.Helper()

	got, err := dictionary.Search(word)
	if err != nil {
		t.Fatal("should find added word:", err)
	}
	assertStrings(t, got, definition)
}
```

我们为 word 和 definition 创建了变量，并把 definition 的断言移到了它自己的辅助函数里。

我们的 `Add` 看起来不错。但是，我们没考虑当我们尝试添加的值已经存在时会怎样！

如果 value 已经存在，map 不会抛错。相反，它会直接用新提供的值覆盖原值。这在实际中可能很方便，但让我们的函数名变得不太准确。`Add` 不应该修改已有的 value，它只应该往字典里添加新单词。

## 先写测试

```go
func TestAdd(t *testing.T) {
	t.Run("new word", func(t *testing.T) {
		dictionary := Dictionary{}
		word := "test"
		definition := "this is just a test"

		err := dictionary.Add(word, definition)

		assertError(t, err, nil)
		assertDefinition(t, dictionary, word, definition)
	})

	t.Run("existing word", func(t *testing.T) {
		word := "test"
		definition := "this is just a test"
		dictionary := Dictionary{word: definition}
		err := dictionary.Add(word, "new test")

		assertError(t, err, ErrWordExists)
		assertDefinition(t, dictionary, word, definition)
	})
}
```

在这个测试中，我们修改了 `Add` 让它返回一个错误，并和一个新的错误变量 `ErrWordExists` 做对比。我们还修改了之前的测试来检查错误是 `nil`。

## 尝试运行测试

编译器会失败，因为我们没有为 `Add` 返回值。

```
./dictionary_test.go:30:13: dictionary.Add(word, definition) used as value
./dictionary_test.go:41:13: dictionary.Add(word, "new test") used as value
```

## 写最少量的代码让测试运行起来，并检查输出

在 `dictionary.go` 中：

```go
var (
	ErrNotFound   = errors.New("could not find the word you were looking for")
	ErrWordExists = errors.New("cannot add word because it already exists")
)

func (d Dictionary) Add(word, definition string) error {
	d[word] = definition
	return nil
}
```

现在我们多了两个错误。我们仍然在修改 value，并且返回了 `nil` 错误。

```
dictionary_test.go:43: got error '%!q(<nil>)' want 'cannot add word because it already exists'
dictionary_test.go:44: got 'new test' want 'this is just a test'
```

## 写足够的代码让测试通过

```go
func (d Dictionary) Add(word, definition string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		d[word] = definition
	case nil:
		return ErrWordExists
	default:
		return err
	}

	return nil
}
```

这里我们用 `switch` 语句来匹配错误。像这样使用 `switch` 提供了一个额外的安全网，以防 `Search` 返回 `ErrNotFound` 之外的错误。

## 重构

我们没什么要重构的，但随着错误使用变多，我们可以做一些调整。

```go
const (
	ErrNotFound   = DictionaryErr("could not find the word you were looking for")
	ErrWordExists = DictionaryErr("cannot add word because it already exists")
)

type DictionaryErr string

func (e DictionaryErr) Error() string {
	return string(e)
}
```

我们让错误成为常量；这要求我们创建自己的 `DictionaryErr` 类型，它实现了 `error` 接口。你可以在 [Dave Cheney 这篇优秀的文章](https://dave.cheney.net/2016/04/07/constant-errors)里读到更多细节。简而言之，这让错误更可复用、更不可变。

接下来，让我们创建一个 `Update` 函数来更新单词的定义。

## 先写测试

```go
func TestUpdate(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{word: definition}
	newDefinition := "new definition"

	dictionary.Update(word, newDefinition)

	assertDefinition(t, dictionary, word, newDefinition)
}
```

`Update` 与 `Add` 关系非常密切，将是我们下一个要实现的功能。

## 尝试运行测试

```
./dictionary_test.go:53:2: dictionary.Update undefined (type Dictionary has no field or method Update)
```

## 写最少量的代码让测试运行起来，并检查失败的输出

我们已经知道怎么处理这种错误。我们需要定义这个函数。

```go
func (d Dictionary) Update(word, definition string) {}
```

加上之后我们能看到，需要把单词的定义改掉。

```
dictionary_test.go:55: got 'this is just a test' want 'new definition'
```

## 写足够的代码让测试通过

我们在修复 `Add` 的问题时已经看到怎么做。所以让我们实现一个和 `Add` 非常相似的版本。

```go
func (d Dictionary) Update(word, definition string) {
	d[word] = definition
}
```

这里没什么需要重构的，因为改动很简单。然而，我们现在遇到了和 `Add` 一样的问题。如果传入一个新单词，`Update` 会把它加到字典里。

## 先写测试

```go
t.Run("existing word", func(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{word: definition}
	newDefinition := "new definition"

	err := dictionary.Update(word, newDefinition)

	assertError(t, err, nil)
	assertDefinition(t, dictionary, word, newDefinition)
})

t.Run("new word", func(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{}

	err := dictionary.Update(word, definition)

	assertError(t, err, ErrWordDoesNotExist)
})
```

我们又添加了一个错误类型，用于单词不存在的情况。我们也修改了 `Update` 让它返回一个 `error` 值。

## 尝试运行测试

```
./dictionary_test.go:53:16: dictionary.Update(word, newDefinition) used as value
./dictionary_test.go:64:16: dictionary.Update(word, definition) used as value
./dictionary_test.go:66:23: undefined: ErrWordDoesNotExist
```

这次我们得到了 3 个错误，但我们已经知道怎么处理了。

## 写最少量的代码让测试运行起来，并检查失败的输出

```go
const (
	ErrNotFound         = DictionaryErr("could not find the word you were looking for")
	ErrWordExists       = DictionaryErr("cannot add word because it already exists")
	ErrWordDoesNotExist = DictionaryErr("cannot perform operation on word because it does not exist")
)

func (d Dictionary) Update(word, definition string) error {
	d[word] = definition
	return nil
}
```

我们添加了自己的错误类型，并返回了 `nil` 错误。

经过这些改动，我们现在能看到一条非常清晰的错误：

```
dictionary_test.go:66: got error '%!q(<nil>)' want 'cannot update word because it does not exist'
```

## 写足够的代码让测试通过

```go
func (d Dictionary) Update(word, definition string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		return ErrWordDoesNotExist
	case nil:
		d[word] = definition
	default:
		return err
	}

	return nil
}
```

这个函数看起来几乎和 `Add` 一样，区别只是我们调换了"何时更新 `dictionary`"和"何时返回错误"。

### 关于为 Update 声明新错误的说明

我们可以复用 `ErrNotFound` 而不必新加一个错误。然而，针对更新失败的情况有一个明确的错误通常更好。

具体的错误能给你更多关于哪里出问题的信息。来看一个 Web 应用的例子：

> 当遇到 `ErrNotFound` 时你可以重定向用户，但遇到 `ErrWordDoesNotExist` 时你可以显示一条错误消息。

接下来，让我们创建一个函数来 `Delete` 字典中的单词。

## 先写测试

```go
func TestDelete(t *testing.T) {
	word := "test"
	dictionary := Dictionary{word: "test definition"}

	dictionary.Delete(word)

	_, err := dictionary.Search(word)
	assertError(t, err, ErrNotFound)
}
```

我们的测试创建了一个含有某个单词的 `Dictionary`，然后检查这个单词是否被移除了。

## 尝试运行测试

运行 `go test` 我们得到：

```
./dictionary_test.go:74:6: dictionary.Delete undefined (type Dictionary has no field or method Delete)
```

## 写最少量的代码让测试运行起来，并检查失败的输出

```go
func (d Dictionary) Delete(word string) {

}
```

加上这个之后，测试告诉我们没有删除该单词。

```
dictionary_test.go:78: got error '%!q(<nil>)' want 'could not find the word you were looking for'
```

## 写足够的代码让测试通过

```go
func (d Dictionary) Delete(word string) {
	delete(d, word)
}
```

Go 有一个内置的 `delete` 函数，可作用于 map。它接受两个参数，没有返回值。第一个参数是 map，第二个是要被移除的 key。

## 重构
没什么要重构的，但我们可以把 `Update` 的同样逻辑用上，处理单词不存在的情形。

```go
func TestDelete(t *testing.T) {
	t.Run("existing word", func(t *testing.T) {
		word := "test"
		dictionary := Dictionary{word: "test definition"}

		err := dictionary.Delete(word)

		assertError(t, err, nil)

		_, err = dictionary.Search(word)

		assertError(t, err, ErrNotFound)
	})

	t.Run("non-existing word", func(t *testing.T) {
		word := "test"
		dictionary := Dictionary{}

		err := dictionary.Delete(word)

		assertError(t, err, ErrWordDoesNotExist)
	})
}
```

## 尝试运行测试

编译器会失败，因为我们没有为 `Delete` 返回值。

```
./dictionary_test.go:77:10: dictionary.Delete(word) (no value) used as value
./dictionary_test.go:90:10: dictionary.Delete(word) (no value) used as value
```

## 写足够的代码让测试通过

```go
func (d Dictionary) Delete(word string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		return ErrWordDoesNotExist
	case nil:
		delete(d, word)
	default:
		return err
	}

	return nil
}
```

我们再次用 switch 语句来匹配错误，处理尝试删除一个不存在单词时的情况。

## 总结

在这一节，我们涉及了很多内容。我们为字典做了一套完整的 CRUD（Create、Read、Update 和 Delete）API。这一过程中我们学习了如何：

* 创建 map
* 在 map 中查找元素
* 往 map 中添加新元素
* 更新 map 中的元素
* 从 map 中删除元素
* 进一步了解错误
  * 如何创建常量错误
  * 编写错误包装类型
