# 指针 & 错误

[**本章的所有代码可以在这里找到**](https://github.com/quii/learn-go-with-tests/tree/main/pointers)

我们在上一节里学习了结构体，它让我们可以把围绕一个概念的若干值聚合起来。

在某个时候，你可能希望使用结构体来管理状态，对外暴露一些方法，让用户能以你可控的方式改变状态。

**金融科技喜爱 Go**，呃还有比特币？所以我们来展示一下，我们能做出多么了不起的银行系统。

我们做一个 `Wallet` 结构体，让我们能够存入 `Bitcoin`。

## 先写测试

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(10)

	got := wallet.Balance()
	want := 10

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

在 [上一个例子](structs-methods-and-interfaces.md) 中，我们直接用字段名访问字段，但是在我们这个 _非常安全的钱包_ 里，我们不希望把内部状态暴露给外界。我们想通过方法来控制访问。

## 尝试运行测试

`./wallet_test.go:7:12: undefined: Wallet`

## 写最少的代码让测试能运行，并检查失败的测试输出

编译器不知道 `Wallet` 是什么，那就告诉它。

```go
type Wallet struct{}
```

现在我们做好了钱包，再尝试运行测试

```
./wallet_test.go:9:8: wallet.Deposit undefined (type Wallet has no field or method Deposit)
./wallet_test.go:11:15: wallet.Balance undefined (type Wallet has no field or method Balance)
```

我们需要定义这些方法。

记住，只做让测试能跑起来所需的事情。我们需要确保我们的测试以清晰的错误消息正确地失败。

```go
func (w Wallet) Deposit(amount int) {

}

func (w Wallet) Balance() int {
	return 0
}
```

如果这个语法不熟悉，回去读一下结构体那一节。

测试现在应该能编译并运行了

`wallet_test.go:15: got 0 want 10`

## 写足够的代码让测试通过

我们需要在结构体里有某种 _balance_ 变量来存储状态

```go
type Wallet struct {
	balance int
}
```

在 Go 中，如果一个符号（变量、类型、函数等）以小写字母开头，那么它在 _定义它的包之外_ 是私有的。

在我们的例子里，我们希望我们的方法能够操作这个值，但其他人不行。

记得我们可以使用"接收器"变量来访问结构体的内部 `balance` 字段。

```go
func (w Wallet) Deposit(amount int) {
	w.balance += amount
}

func (w Wallet) Balance() int {
	return w.balance
}
```

我们在金融科技的事业有了保障，运行测试套件，享受测试通过

`wallet_test.go:15: got 0 want 10`

### 这不太对劲

嗯这令人困惑，我们的代码看起来应该能工作。我们把新的金额加到 balance 上，然后 balance 方法应该返回它的当前状态。

在 Go 中，**当你调用一个函数或方法时，参数是被** _**复制**_ **的**。

调用 `func (w Wallet) Deposit(amount int)` 时，`w` 是我们调用方法时所在对象的副本。

不太学究地说，当你创建一个值 —— 比如一个 wallet —— 它会被存储在内存的某个位置。你可以通过 `&myVal` 找到那块内存的 _地址_。

通过往代码里加一些打印来做实验

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(10)

	got := wallet.Balance()

	fmt.Printf("address of balance in test is %p \n", &wallet.balance)

	want := 10

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

```go
func (w Wallet) Deposit(amount int) {
	fmt.Printf("address of balance in Deposit is %p \n", &w.balance)
	w.balance += amount
}
```

`%p` 占位符以十六进制带 `0x` 前缀的形式打印内存地址，转义字符则打印一个换行。注意，我们通过在符号前面加 `&` 字符来获取它的指针（内存地址）。

现在重新运行测试

```
address of balance in Deposit is 0xc420012268
address of balance in test is 0xc420012260
```

你可以看到两个 balance 的地址是不同的。所以当我们在代码里改变 balance 的值时，我们是在操作来自测试的副本。因此测试中的 balance 没有改变。

我们可以用 _指针_ 来修复这个问题。[指针](https://gobyexample.com/pointers) 让我们能 _指向_ 某些值，然后让我们能够改变它们。所以我们不取整个 Wallet 的副本，而是取那个 wallet 的指针，这样我们就能改变它内部的原始值。

```go
func (w *Wallet) Deposit(amount int) {
	w.balance += amount
}

func (w *Wallet) Balance() int {
	return w.balance
}
```

差别在于接收器的类型是 `*Wallet` 而不是 `Wallet`，你可以读作"指向一个 wallet 的指针"。

试着重新运行测试，它们应该可以通过了。

现在你可能会问，为什么它们能通过？我们没有像下面这样在函数里解引用指针：

```go
func (w *Wallet) Balance() int {
	return (*w).balance
}
```

而是看起来直接寻址了对象。事实上，上面用 `(*w)` 的代码是完全有效的。然而 Go 的设计者认为这种记法很麻烦，所以语言允许我们写 `w.balance`，不需要显式的解引用。这些指向结构体的指针甚至有自己的名字：_结构体指针_，并且它们是 [自动解引用的](https://golang.org/ref/spec#Method_values)。

技术上你不需要改 `Balance` 用指针接收器，因为取 balance 的副本也没问题。然而按照惯例，为了一致性，你应该让方法的接收器类型保持一致。

## 重构

我们说我们要做一个比特币钱包，但目前为止还没提到它。我们一直在用 `int`，因为它是一个适合计数的好类型！

为此创建一个 `struct` 似乎有点过头。`int` 在工作方式上没问题，但它没有描述性。

Go 让你可以从已有的类型创建新类型。

语法是 `type MyName OriginalType`

```go
type Bitcoin int

type Wallet struct {
	balance Bitcoin
}

func (w *Wallet) Deposit(amount Bitcoin) {
	w.balance += amount
}

func (w *Wallet) Balance() Bitcoin {
	return w.balance
}
```

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(Bitcoin(10))

	got := wallet.Balance()

	want := Bitcoin(10)

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

要创建 `Bitcoin`，你只需用语法 `Bitcoin(999)`。

通过这样做，我们创建了一个新类型，并可以在它们上声明 _方法_。当你想在已有类型之上添加一些领域特定的功能时，这会非常有用。

让我们在 Bitcoin 上实现 [Stringer](https://golang.org/pkg/fmt/#Stringer)

```go
type Stringer interface {
	String() string
}
```

这个接口定义在 `fmt` 包中，让你可以定义当你的类型在打印时与 `%s` 格式化字符串一起使用时如何被打印。

```go
func (b Bitcoin) String() string {
	return fmt.Sprintf("%d BTC", b)
}
```

如你所见，在类型声明上创建方法的语法和在结构体上的语法是一样的。

接下来我们需要更新测试中的格式化字符串，让它们使用 `String()`。

```go
if got != want {
	t.Errorf("got %s want %s", got, want)
}
```

为了看到这个效果，故意把测试弄坏，这样我们就能看到它

`wallet_test.go:18: got 10 BTC want 20 BTC`

这让我们的测试中发生了什么变得更清晰。

下一个需求是 `Withdraw` 函数。

## 先写测试

基本上是 `Deposit()` 的反向

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}

		wallet.Deposit(Bitcoin(10))

		got := wallet.Balance()

		want := Bitcoin(10)

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	})

	t.Run("withdraw", func(t *testing.T) {
		wallet := Wallet{balance: Bitcoin(20)}

		wallet.Withdraw(Bitcoin(10))

		got := wallet.Balance()

		want := Bitcoin(10)

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	})
}
```

## 尝试运行测试

`./wallet_test.go:26:9: wallet.Withdraw undefined (type Wallet has no field or method Withdraw)`

## 写最少的代码让测试能运行，并检查失败的测试输出

```go
func (w *Wallet) Withdraw(amount Bitcoin) {

}
```

`wallet_test.go:33: got 20 BTC want 10 BTC`

## 写足够的代码让测试通过

```go
func (w *Wallet) Withdraw(amount Bitcoin) {
	w.balance -= amount
}
```

## 重构

我们的测试里有一些重复，把它重构出来。

```go
func TestWallet(t *testing.T) {

	assertBalance := func(t testing.TB, wallet Wallet, want Bitcoin) {
		t.Helper()
		got := wallet.Balance()

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	}

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw", func(t *testing.T) {
		wallet := Wallet{balance: Bitcoin(20)}
		wallet.Withdraw(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

}
```

如果你试图 `Withdraw` 比账户里剩下的更多金额，应该发生什么？目前我们的需求是假设没有透支机制。

我们如何在使用 `Withdraw` 时表示出现问题？

在 Go 中，如果你想表示一个错误，惯用的做法是让你的函数返回一个 `err`，让调用者去检查并采取行动。

我们在测试里试一下这个。

## 先写测试

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertBalance(t, wallet, startingBalance)

	if err == nil {
		t.Error("wanted an error but didn't get one")
	}
})
```

我们希望 `Withdraw` 在你尝试取出比你拥有的更多金额时返回一个错误，而 balance 应保持不变。

然后我们检查是否返回了错误，如果它是 `nil` 则让测试失败。

`nil` 等同于其他编程语言的 `null`。错误可以是 `nil`，因为 `Withdraw` 的返回类型是 `error`，它是一个接口。如果你看到一个函数接受参数或返回的值是接口，它们就可以是 nil。

像 `null` 一样，如果你尝试访问一个 `nil` 的值，它会抛出一个**运行时 panic**。这很糟！你应该确保检查 nil。

## 尝试运行测试

`./wallet_test.go:31:25: wallet.Withdraw(Bitcoin(100)) used as value`

措辞可能有点不清楚，但我们之前对 `Withdraw` 的意图只是调用它，它永远不会返回值。要让它编译，我们需要把它改成有返回类型。

## 写最少的代码让测试能运行，并检查失败的测试输出

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {
	w.balance -= amount
	return nil
}
```

再次强调，只写满足编译器所需的代码非常重要。我们更正了 `Withdraw` 方法返回 `error`，目前我们必须返回 _某个东西_，所以就返回 `nil`。

## 写足够的代码让测试通过

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return errors.New("oh no")
	}

	w.balance -= amount
	return nil
}
```

记得把 `errors` 导入到你的代码里。

`errors.New` 创建一个带有你选择的消息的新 `error`。

## 重构

让我们快速做一个测试辅助函数来检查错误，提升测试的可读性

```go
assertError := func(t testing.TB, err error) {
	t.Helper()
	if err == nil {
		t.Error("wanted an error but didn't get one")
	}
}
```

然后在我们的测试里

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertError(t, err)
	assertBalance(t, wallet, startingBalance)
})
```

希望当返回 "oh no" 这个错误时你在想我们 _可能_ 会迭代它，因为它看起来不太有用。

假设错误最终会返回给用户，让我们更新测试，断言某种错误消息，而不只是断言错误存在。

## 先写测试

更新辅助函数让它接受一个 `string` 用于比较。

```go
assertError := func(t testing.TB, got error, want string) {
	t.Helper()

	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got.Error() != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

如你所见，`Error` 可以通过 `.Error()` 方法转换为字符串，我们这样做是为了把它和我们想要的字符串进行比较。我们还确保错误不是 `nil`，以确保我们不会在 `nil` 上调用 `.Error()`。

然后更新调用方

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertError(t, err, "cannot withdraw, insufficient funds")
	assertBalance(t, wallet, startingBalance)
})
```

我们引入了 `t.Fatal`，它在被调用时会停止测试。这是因为如果没有错误，我们不想再对返回的错误做任何更多的断言。否则的话，测试会继续走到下一步并因为空指针而 panic。

## 尝试运行测试

`wallet_test.go:61: got err 'oh no' want 'cannot withdraw, insufficient funds'`

## 写足够的代码让测试通过

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return errors.New("cannot withdraw, insufficient funds")
	}

	w.balance -= amount
	return nil
}
```

## 重构

我们在测试代码和 `Withdraw` 代码中都重复了错误消息。

如果有人想重新措辞这个错误，导致测试失败，那会很烦人，而且这对我们的测试来说细节太多了。我们 _并不真的_ 关心确切的措辞，只关心在某种条件下会返回某种关于取款的有意义的错误。

在 Go 中，错误是值，所以我们可以把它重构成一个变量，作为它的单一事实来源。

```go
var ErrInsufficientFunds = errors.New("cannot withdraw, insufficient funds")

func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return ErrInsufficientFunds
	}

	w.balance -= amount
	return nil
}
```

`var` 关键字让我们可以定义对包来说是全局的值。

这本身就是一个积极的改变，因为现在我们的 `Withdraw` 函数看起来非常清晰。

接下来我们可以重构测试代码，使用这个值代替具体的字符串。

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw with funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		wallet.Withdraw(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw insufficient funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(100))

		assertError(t, err, ErrInsufficientFunds)
		assertBalance(t, wallet, Bitcoin(20))
	})
}

func assertBalance(t testing.TB, wallet Wallet, want Bitcoin) {
	t.Helper()
	got := wallet.Balance()

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}

func assertError(t testing.TB, got, want error) {
	t.Helper()
	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

现在测试也更易跟随了。

我把辅助函数移到主测试函数之外，这样当有人打开文件时，他们可以先开始读我们的断言，而不是一些辅助函数。

测试的另一个有用的属性是它们帮我们理解我们代码的 _实际_ 用法，这样我们就能写出体贴的代码。我们这里可以看到，开发者可以简单地调用我们的代码，对 `ErrInsufficientFunds` 做相等检查并相应行动。

### 未检查的错误

虽然 Go 编译器在很多方面帮了你大忙，但有时还是有一些你可能漏掉的事情，错误处理有时会很棘手。

有一个场景我们没有测试。要找到它，在终端里运行下面的命令安装 `errcheck`，它是 Go 中诸多 linter 之一。

`go install github.com/kisielk/errcheck@latest`

然后，在你代码所在的目录里运行 `errcheck .`

你应该会得到类似这样的输出

`wallet_test.go:17:18: wallet.Withdraw(Bitcoin(10))`

它告诉我们的是，我们没有检查那一行代码上返回的错误。在我电脑上那一行代码对应我们正常的取款场景，因为我们没有检查 `Withdraw` 成功时是否 _没有_ 返回错误。

这是考虑到这一点的最终测试代码。

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))

		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw with funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(10))

		assertNoError(t, err)
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw insufficient funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(100))

		assertError(t, err, ErrInsufficientFunds)
		assertBalance(t, wallet, Bitcoin(20))
	})
}

func assertBalance(t testing.TB, wallet Wallet, want Bitcoin) {
	t.Helper()
	got := wallet.Balance()

	if got != want {
		t.Errorf("got %s want %s", got, want)
	}
}

func assertNoError(t testing.TB, got error) {
	t.Helper()
	if got != nil {
		t.Fatal("got an error but didn't want one")
	}
}

func assertError(t testing.TB, got error, want error) {
	t.Helper()
	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got != want {
		t.Errorf("got %s, want %s", got, want)
	}
}
```

## 总结

### 指针

* Go 在你把值传给函数/方法时会复制它们，所以如果你写一个需要改变状态的函数，你需要它接受一个指向你想改变的东西的指针。
* Go 复制值这一事实在很多时候是有用的，但有时你不希望系统复制某样东西，这种情况下你需要传引用。例子包括引用非常大的数据结构或仅需要一个实例的东西（比如数据库连接池）。

### nil

* 指针可以是 nil
* 当一个函数返回某样东西的指针时，你需要确保检查它是不是 nil，否则可能引发运行时异常 —— 编译器在这里帮不了你。
* 适合用来描述可能缺失的值

### 错误

* 错误是调用函数/方法时表示失败的方式。
* 通过倾听我们的测试，我们得出在错误中检查字符串会导致不稳定测试的结论。所以我们重构了实现，改用一个有意义的值，这导致代码更易测试，也得出这对我们 API 的用户来说也会更容易的结论。
* 这并不是错误处理的全部故事，你可以做更复杂的事情，但这只是一个介绍。后面的章节将涵盖更多策略。
* [不要只是检查错误，要优雅地处理它们](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

### 从已有类型创建新类型

* 适合给值添加更多领域特定的含义
* 可以让你实现接口

指针和错误是写 Go 时一个重要的部分，你需要熟悉它们。值得欣慰的是，如果你做错了什么，编译器 _通常_ 会帮你，你只需要花点时间读读错误信息。
