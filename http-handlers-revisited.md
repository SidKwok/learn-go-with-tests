# 重看 HTTP Handler

[**本章的所有代码可以在这里找到**](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/http-handlers-revisited)

本书已经有一章关于[测试 HTTP handler](http-server.md)的内容，但本章会更广泛地讨论如何设计它们，让它们便于测试。

我们会看一个真实的例子，并通过应用单一职责原则和关注点分离等原则来改进它的设计。这些原则可以通过使用[接口](structs-methods-and-interfaces.md)和[依赖注入](dependency-injection.md)来实现。这样我们就会展示测试 handler 其实相当 trivial。

![Go 社区中常见问题的图示](.gitbook/assets/amazing-art.png)

测试 HTTP handler 似乎是 Go 社区中反复被问到的问题，我认为这指向一个更广泛的问题：人们误解了如何设计它们。

人们在测试上的困难往往源于代码的设计而非编写测试本身。正如我在本书中反复强调的：

> 如果你的测试让你痛苦，听从那个信号，思考一下你代码的设计。

## 一个例子

[Santosh Kumar 在推特上问我](https://twitter.com/sntshk/status/1255559003339284481)

> 我怎么测试一个有 mongodb 依赖的 http handler？

下面是代码

```go
func Registration(w http.ResponseWriter, r *http.Request) {
	var res model.ResponseResult
	var user model.User

	w.Header().Set("Content-Type", "application/json")

	jsonDecoder := json.NewDecoder(r.Body)
	jsonDecoder.DisallowUnknownFields()
	defer r.Body.Close()

	// check if there is proper json body or error
	if err := jsonDecoder.Decode(&user); err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// Connect to mongodb
	client, _ := mongo.NewClient(options.Client().ApplyURI("mongodb://127.0.0.1:27017"))
	ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
	err := client.Connect(ctx)
	if err != nil {
		panic(err)
	}
	defer client.Disconnect(ctx)
	// Check if username already exists in users datastore, if so, 400
	// else insert user right away
	collection := client.Database("test").Collection("users")
	filter := bson.D{{"username", user.Username}}
	var foundUser model.User
	err = collection.FindOne(context.TODO(), filter).Decode(&foundUser)
	if foundUser.Username == user.Username {
		res.Error = UserExists
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	pass, err := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
	if err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}
	user.Password = string(pass)

	insertResult, err := collection.InsertOne(context.TODO(), user)
	if err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// return 200
	w.WriteHeader(http.StatusOK)
	res.Result = fmt.Sprintf("%s: %s", UserCreated, insertResult.InsertedID)
	json.NewEncoder(w).Encode(res)
	return
}
```

我们来列一下这一个函数要做的所有事情：

1. 写 HTTP 响应、发送响应头、状态码等。
2. 把请求体解码成一个 `User`
3. 连接数据库（以及围绕这件事的所有细节）
4. 查询数据库并根据结果应用一些业务逻辑
5. 生成密码
6. 插入一条记录

这太多了。

## 什么是 HTTP Handler，它应该做什么？

暂时忘掉 Go 的具体细节，无论我用过哪种语言，一直对我有帮助的是思考[关注点分离](https://en.wikipedia.org/wiki/Separation_of_concerns)和[单一职责原则](https://en.wikipedia.org/wiki/Single-responsibility_principle)。

应用这些原则可能相当棘手，取决于你要解决的问题。一个职责到底 _是_ 什么？

界限可能因你思考的抽象层次而变得模糊，有时你的第一直觉可能不对。

幸运的是，对于 HTTP handler，我有比较清晰的想法，无论我做的是什么项目：

1. 接受一个 HTTP 请求，解析并校验它。
2. 调用某个 `ServiceThing`，用从第 1 步得到的数据去做 `ImportantBusinessLogic`。
3. 根据 `ServiceThing` 的返回结果发送合适的 `HTTP` 响应。

我不是说 _所有_ HTTP handler 都应该大致是这种形状，但 99 次中有 99 次对我来说似乎都是这种情况。

当你像这样分离这些关注点：

* 测试 handler 变得轻而易举，并聚焦于少数几个关注点。
* 重要的是，测试 `ImportantBusinessLogic` 不再需要关心 `HTTP`，你可以干净地测试业务逻辑。
* 你可以在其他场景中使用 `ImportantBusinessLogic`，而不必修改它。
* 如果 `ImportantBusinessLogic` 的行为发生变化，只要接口保持不变，你就不必修改你的 handler。

## Go 的 Handler

[`http.HandlerFunc`](https://golang.org/pkg/net/http/#HandlerFunc)

> HandlerFunc 类型是一个适配器，允许把普通函数用作 HTTP handler。

`type HandlerFunc func(ResponseWriter, *Request)`

读者，深呼吸看一眼上面的代码。你注意到什么了？

**它就是一个接受一些参数的函数**

没有框架魔法、没有注解、没有魔豆，什么都没有。

它就是一个函数，_而我们知道怎么测试函数_。

它和上面的描述很契合：

* 它接受一个 [`http.Request`](https://golang.org/pkg/net/http/#Request)，这只是一捆数据，让我们检查、解析和校验。
* > [HTTP handler 使用 `http.ResponseWriter` 接口来构造 HTTP 响应。](https://golang.org/pkg/net/http/#ResponseWriter)

### 超基础的示例测试

```go
func Teapot(res http.ResponseWriter, req *http.Request) {
	res.WriteHeader(http.StatusTeapot)
}

func TestTeapotHandler(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/", nil)
	res := httptest.NewRecorder()

	Teapot(res, req)

	if res.Code != http.StatusTeapot {
		t.Errorf("got status %d but wanted %d", res.Code, http.StatusTeapot)
	}
}
```

要测试我们的函数，我们 _调用_ 它。

在测试里，我们传入 `httptest.ResponseRecorder` 作为 `http.ResponseWriter` 参数，我们的函数会用它来写 `HTTP` 响应。这个 recorder 会记录（或者说 _spy_ 监视）发送了什么，然后我们就能做断言。

## 在我们的 handler 中调用 `ServiceThing`

人们常见的对 TDD 教程的抱怨是它们总是"太简单"且不够"贴近真实世界"。我对此的回答是：

> 如果你所有的代码都像你提到的那些例子一样易读易测，不是很好吗？

这是我们面临的最大挑战之一，但需要继续努力追求。如果我们练习并应用良好的软件工程原则，设计出易读易测的代码 _是可能的_（虽然不一定容易）。

回顾一下前面那个 handler 做了什么：

1. 写 HTTP 响应、发送响应头、状态码等。
2. 把请求体解码成一个 `User`
3. 连接数据库（以及围绕这件事的所有细节）
4. 查询数据库并根据结果应用一些业务逻辑
5. 生成密码
6. 插入一条记录

按照更理想的关注点分离的思路，我希望它更像这样：

1. 把请求体解码成一个 `User`
2. 调用 `UserService.Register(user)`（这是我们的 `ServiceThing`）
3. 如果有错误就采取相应行动（这个例子总是发送 `400 BadRequest`，我觉得不太对），_目前_ 我会用一个兜底的 `500 Internal Server Error` handler。我必须强调，对所有错误返回 `500` 会造成一个糟糕的 API！稍后我们可以让错误处理更精细，也许使用[错误类型](error-types.md)。
4. 如果没有错误，返回 `201 Created`，并把 ID 作为响应体（同样为了简洁/偷懒）

为了简洁，我不会再走一遍常规的 TDD 过程，可参考其他章节的例子。

### 新设计

```go
type UserService interface {
	Register(user User) (insertedID string, err error)
}

type UserServer struct {
	service UserService
}

func NewUserServer(service UserService) *UserServer {
	return &UserServer{service: service}
}

func (u *UserServer) RegisterUser(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()

	// 请求解析和校验
	var newUser User
	err := json.NewDecoder(r.Body).Decode(&newUser)

	if err != nil {
		http.Error(w, fmt.Sprintf("could not decode user payload: %v", err), http.StatusBadRequest)
		return
	}

	// 调用一个 service 来处理重活
	insertedID, err := u.service.Register(newUser)

	// 根据返回结果，相应地响应
	if err != nil {
		//todo: 不同种类的错误以不同方式处理
		http.Error(w, fmt.Sprintf("problem registering new user: %v", err), http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusCreated)
	fmt.Fprint(w, insertedID)
}
```

我们的 `RegisterUser` 方法符合 `http.HandlerFunc` 的形状，所以可以用了。我们把它作为一个新类型 `UserServer` 上的方法附加，这个类型对 `UserService` 的依赖被捕获为一个接口。

接口是确保我们的 `HTTP` 关注点与任何具体实现解耦的极好方式；我们只需调用依赖上的方法，不必关心 _用户是怎样_ 注册的。

如果你希望以 TDD 的方式更详细地探索这种方法，请阅读[依赖注入](dependency-injection.md)章节，以及["构建一个应用"部分中的 HTTP 服务器章节](http-server.md)。

既然我们已经把自己与具体的注册实现细节解耦，编写 handler 代码就非常直接，并遵循前面描述的职责。

### 测试！

这种简单性也反映在我们的测试里。

```go
type MockUserService struct {
	RegisterFunc    func(user User) (string, error)
	UsersRegistered []User
}

func (m *MockUserService) Register(user User) (insertedID string, err error) {
	m.UsersRegistered = append(m.UsersRegistered, user)
	return m.RegisterFunc(user)
}

func TestRegisterUser(t *testing.T) {
	t.Run("can register valid users", func(t *testing.T) {
		user := User{Name: "CJ"}
		expectedInsertedID := "whatever"

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return expectedInsertedID, nil
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusCreated)

		if res.Body.String() != expectedInsertedID {
			t.Errorf("expected body of %q but got %q", res.Body.String(), expectedInsertedID)
		}

		if len(service.UsersRegistered) != 1 {
			t.Fatalf("expected 1 user added but got %d", len(service.UsersRegistered))
		}

		if !reflect.DeepEqual(service.UsersRegistered[0], user) {
			t.Errorf("the user registered %+v was not what was expected %+v", service.UsersRegistered[0], user)
		}
	})

	t.Run("returns 400 bad request if body is not valid user JSON", func(t *testing.T) {
		server := NewUserServer(nil)

		req := httptest.NewRequest(http.MethodGet, "/", strings.NewReader("trouble will find me"))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusBadRequest)
	})

	t.Run("returns a 500 internal server error if the service fails", func(t *testing.T) {
		user := User{Name: "CJ"}

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return "", errors.New("couldn't add new user")
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusInternalServerError)
	})
}
```

现在我们的 handler 不再耦合到具体的存储实现，写一个 `MockUserService` 来帮我们写简单、快速的单元测试以验证它特定的职责，就变得轻松了。

### 那数据库代码呢？你在偷懒！

这都是有意为之的。我们不希望 HTTP handler 关心我们的业务逻辑、数据库、连接等等。

通过这样做，我们把 handler 从混乱的细节中解放出来，我们 _也_ 让测试持久化层和业务逻辑变得更容易，因为它们也不再耦合到无关的 HTTP 细节。

我们现在所要做的就是用任何我们想用的数据库实现 `UserService`

```go
type MongoUserService struct {
}

func NewMongoUserService() *MongoUserService {
	//todo: 把 DB URL 作为参数传给这个函数
	//todo: 连接 db，创建一个连接池
	return &MongoUserService{}
}

func (m MongoUserService) Register(user User) (insertedID string, err error) {
	// 用 m.mongoConnection 来执行查询
	panic("implement me")
}
```

我们可以单独测试这个，一旦满意了，在 `main` 中我们可以把这两个单元拼起来，得到能工作的应用。

```go
func main() {
	mongoService := NewMongoUserService()
	server := NewUserServer(mongoService)
	http.ListenAndServe(":8000", http.HandlerFunc(server.RegisterUser))
}
```

### 用很少的努力得到更健壮、更可扩展的设计

这些原则不仅在短期让我们的生活更轻松，也让系统在未来更容易扩展。

如果系统的后续迭代希望给用户发一封注册确认邮件，我们应该不会感到惊讶。

按照旧设计，我们必须修改 handler _以及_ 周围的测试。这就是为什么部分代码会变得无法维护——越来越多的功能蔓延进来，因为它本来就是 _那样_ 设计的；让"HTTP handler"处理……一切！

通过用接口分离关注点，我们 _完全_ 不必修改 handler，因为它不关心围绕注册的业务逻辑。

## 总结

测试 Go 的 HTTP handler 不难，但设计良好的软件可能很难！

人们犯了一个错误，认为 HTTP handler 是特殊的，写它们时就抛弃了好的软件工程实践，这反过来让测试它们变得困难。

再次重申；**Go 的 http handler 就是函数**。如果你像写其他函数一样写它们——有清晰的职责、良好的关注点分离——你测试它们就不会有麻烦，你的代码库也会因此更健康。
