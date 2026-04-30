# Error types

**[本章的所有代码可以在这里找到](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/error-types)**

**为错误自定义类型可以是一种优雅的整理代码的方式，能让你的代码更易使用、也更易测试。**

Pedro 在 Gopher Slack 上提问

> 如果我用 `fmt.Errorf("%s must be foo, got %s", bar, baz)` 创建了一个错误，有没有办法不通过比较字符串值来测试相等？

我们来虚构一个函数，借此探讨这个想法。

```go
// DumbGetter 在拿到 200 时返回 url 的字符串响应体
func DumbGetter(url string) (string, error) {
	res, err := http.Get(url)

	if err != nil {
		return "", fmt.Errorf("problem fetching from %s, %v", url, err)
	}

	if res.StatusCode != http.StatusOK {
		return "", fmt.Errorf("did not get 200 from %s, got %d", url, res.StatusCode)
	}

	defer res.Body.Close()
	body, _ := io.ReadAll(res.Body) // 为了简洁忽略 err

	return string(body), nil
}
```

写一个可能因不同原因失败的函数并不少见，我们也希望能正确处理每种场景。

正如 Pedro 所说，我们 _可以_ 像下面这样为状态错误写一个测试。

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	want := fmt.Sprintf("did not get 200 from %s, got %d", svr.URL, http.StatusTeapot)
	got := err.Error()

	if got != want {
		t.Errorf(`got "%v", want "%v"`, got, want)
	}
})
```

这个测试创建了一个总是返回 `StatusTeapot` 的服务器，然后用它的 URL 作为 `DumbGetter` 的参数，用来验证它能正确处理非 `200` 的响应。

## 这种测试方式的问题

这本书一直强调 _倾听你的测试_，而这个测试感觉并不好：

- 我们在用与生产代码完全相同的方式构造同一个字符串来测试它
- 它读起来、写起来都很烦
- 这个具体的错误消息字符串真的是我们 _关心的东西_ 吗？

这告诉我们什么？我们测试的"人体工学"会反映到使用我们代码的另一段代码上。

我们代码的使用者会怎么对我们返回的具体错误类型作出反应？他们顶多就是看错误字符串，这极易出错而且写起来也很糟。

## 我们应该怎么做

借助 TDD 我们能进入这样的思维模式：

> _我_ 想怎么使用这段代码？

对 `DumbGetter` 来说，我们可以提供一种方式，让用户通过类型系统来理解发生了什么样的错误。

如果 `DumbGetter` 能返回类似下面这样的东西呢

```go
type BadStatusError struct {
	URL    string
	Status int
}
```

不再是一个魔法字符串，我们有了实际的 _数据_ 可以使用。

我们改造现有的测试来体现这个需求

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	got, isStatusErr := err.(BadStatusError)

	if !isStatusErr {
		t.Fatalf("was not a BadStatusError, got %T", err)
	}

	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

我们需要让 `BadStatusError` 实现 error 接口。

```go
func (b BadStatusError) Error() string {
	return fmt.Sprintf("did not get 200 from %s, got %d", b.URL, b.Status)
}
```

### 这个测试做了什么？

我们没有去检查错误的具体字符串，而是对错误做了一次[类型断言](https://tour.golang.org/methods/15)，看它是不是一个 `BadStatusError`。这更清晰地表达了我们对 _错误类型_ 的关心。如果断言通过，我们再检查错误的属性是否正确。

跑一下测试，它告诉我们没返回正确类型的错误

```
--- FAIL: TestDumbGetter (0.00s)
    --- FAIL: TestDumbGetter/when_you_dont_get_a_200_you_get_a_status_error (0.00s)
    	error-types_test.go:56: was not a BadStatusError, got *errors.errorString
```

我们更新 `DumbGetter` 的错误处理代码使用我们的类型来修复

```go
if res.StatusCode != http.StatusOK {
	return "", BadStatusError{URL: url, Status: res.StatusCode}
}
```

这个改动带来了一些 _实在的好处_

- 我们的 `DumbGetter` 函数变简单了，它不再关心错误字符串的细节，只是创建一个 `BadStatusError`。
- 我们的测试现在反映（也记录了）我们代码的用户在希望做更复杂的错误处理（不止是日志记录）时 _可以怎么做_。只要做一次类型断言，就能很方便地访问错误的属性。
- 它仍然 "只是" 一个 `error`，所以如果他们愿意，也可以像处理任何其他 `error` 一样把它向上抛或者记录日志。

## 总结

如果你发现自己在测试多个错误条件，不要落入比较错误消息的陷阱。

这会导致脆弱、难读难写的测试，也反映出当你代码的使用者需要根据不同错误类型做不同处理时会面临的困境。

始终确保你的测试反映 _你_ 想怎么使用你的代码，从这个角度来看，应当考虑创建错误类型来封装不同种类的错误。这能让代码使用者处理不同类型的错误更容易，也让你的错误处理代码更简洁、更易读。

## 附录

从 Go 1.13 开始，标准库提供了处理错误的新方式，相关内容见 [Go Blog](https://blog.golang.org/go1.13-errors)

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	var got BadStatusError
	isBadStatusError := errors.As(err, &got)
	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if !isBadStatusError {
		t.Fatalf("was not a BadStatusError, got %T", err)
	}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

这里我们用 [`errors.As`](https://pkg.go.dev/errors#example-As) 尝试把错误抽取为我们的自定义类型。它返回一个 `bool` 表示是否成功，并把抽取到的值放到 `got` 里。
