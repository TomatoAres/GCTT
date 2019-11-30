# Go 语言中的集成测试：第二部分 - 设计和编写测试

## 序幕

这篇文章是集成测试系列两个部分中的第二部分。你可以先读 [第一部分：使用 Docker 在有限的环境中执行测试](https://www.ardanlabs.com/blog/2019/03/integration-testing-in-go-executing-tests-with-docker.html)。本文中的示例可以从 [代码仓库](https://github.com/george-e-shaw-iv/integration-tests-example) 获取。

## 简介

> “比起测试行为，设计测试行为是已知的最好的错误预防程序之一。” —— Boris Beizer

在执行集成测试之前，必须正确配置该测试相关的外部系统。否则，测试结果是无效和不可靠的。例如，数据库需要有定义好的数据，这些数据对于要测试的行为是正确的。测试期间更改的数据需要进行验证，尤其是如果要求更改的数据对于后续测试而言是准确的时侯。

Go 测试工具提供了有在执行测试功能前执行代码的能力，使用叫做 `TestMain` 的入口函数实现。它类似于 Go 应用程序的 `Main` 函数。有了 `TestMain` 函数，我们可以在执行测试之前做其他系统配置，比如数据库连接之类的。在本文中，我将分享如何使用它 `TestMain` 来配置和连接 Postgres 数据库，以及如何针对该数据库编写和运行测试。

## 构造填充数据

为了填充数据库，需要定义数据并将其放置在测试工具可以访问的位置。一种常见的方法是定义一个 SQL 文件，该文件是项目的一部分，并且包含所有需要执行的 SQL 命令。另一种方法是将 SQL 命令存储在代码内部的常量中。不同于这两种方法，我将只使用 Go 语言实现来解决此问题。

通常情况下，你已将你的数据结构定义为 Go 结构体类型，用于数据库通信。我将利用这些已存在的数据结构，已经可以控制数据从数据库中流入流出。基于已有的数据结构声明变量，构造所有填充数据，而无需 SQL 语句。

我喜欢这种解决方式，因为它简化了编写集成测试和验证数据是否能够正确用于数据库和应用程序之间的通信的。不必将数据直接与 JSON 比较，就可以将数据解编为适当的类型，然后直接与为之前数据结构定义的变量进行比较。这不仅可以最大程度地减少测试中的语法比较错误，还可以使您的测试更具可维护性、可扩展性和可读性。

## 填充数据库

在我分享的项目中，所有用于填充数数据库功能本都在叫做 [`testdb`](https://github.com/george-e-shaw-iv/integration-tests-example) 的包中。这个包仅用于测试，不用做第三方依赖。用来辅助填充测试数据库的三个主要的函数分别是：`SeedLists`, `SeedItems`, 和 `Truncate`，如下：

这是 `SeedLists` 函数：

### 代码清单 1

```golang
func SeedLists(dbc *sqlx.DB) ([]list.List, error) {
    now := time.Now().Truncate(time.Microsecond)

    lists := []list.List{
        {
            Name:     "Grocery",
            Created:  now,
            Modified: now,
        },
        {
            Name:     "To-do",
            Created:  now,
            Modified: now,
        },
        {
            Name:     "Employees",
            Created:  now,
            Modified: now,
        },
    }

    for i := range lists {
        stmt, err := dbc.Prepare("INSERT INTO list (name, created, modified) VALUES ($1, $2, $3) RETURNING list_id;")
        if err != nil {
            return nil, errors.Wrap(err, "prepare list insertion")
        }

        row := stmt.QueryRow(lists[i].Name, lists[i].Created, lists[i].Modified)

        if err = row.Scan(&lists[i].ID); err != nil {
            if err := stmt.Close(); err != nil {
                return nil, errors.Wrap(err, "close psql statement")
            }

            return nil, errors.Wrap(err, "capture list id")
        }

        if err := stmt.Close(); err != nil {
            return nil, errors.Wrap(err, "close psql statement")
        }
    }

    return lists, nil
}
```

代码清单 1 展示了 `SeedLists` 函数及其如何创建测试数据。在第 35-51 行，list.List 定义了一个用于插入的数据表。然后在 53-72 行，将测试数据插入数据库。为了帮助将插入的数据与测试期间进行的任何数据库调用的结果进行比较，测试数据集在第 74 行返回给调用方。

接下来，我们看看将更多测试数据插入数据库的 `SeedItems` 函数。

### 代码清单 2

```golang
func SeedItems(dbc *sqlx.DB, lists []list.List) ([]item.Item, error) {
    now := time.Now().Truncate(time.Microsecond)

    items := []item.Item{
        {
            ListID:   lists[0].ID, // Grocery
            Name:     "Chocolate Milk",
            Quantity: 1,
            Created:  now,
            Modified: now,
        },
        {
            ListID:   lists[0].ID, // Grocery
            Name:     "Mac and Cheese",
            Quantity: 2,
            Created:  now,
            Modified: now,
        },
        {
            ListID:   lists[1].ID, // To-do
            Name:     "Write Integration Tests",
            Quantity: 1,
            Created:  now,
            Modified: now,
        },
    }

    for i := range items {
        stmt, err := dbc.Prepare("INSERT INTO item (list_id, name, quantity, created, modified) VALUES ($1, $2, $3, $4, $5) RETURNING item_id;")
        if err != nil {
            return nil, errors.Wrap(err, "prepare item insertion")
        }

        row := stmt.QueryRow(items[i].ListID, items[i].Name, items[i].Quantity, items[i].Created, items[i].Modified)

        if err = row.Scan(&items[i].ID); err != nil {
            if err := stmt.Close(); err != nil {
                return nil, errors.Wrap(err, "close psql statement")
            }

            return nil, errors.Wrap(err, "capture list id")
        }

        if err := stmt.Close(); err != nil {
            return nil, errors.Wrap(err, "close psql statement")
        }
    }

    return items, nil
}
```

代码清单 2 显示了 `SeedItems` 函数如何创建测试数据。除了使用 `item.Item` 数据类型，该代码与清单 1 基本相同。`testdb` 包中唯一要共用的函数只有 `Truncate`。

### 代码清单 3

```golang
func Truncate(dbc *sqlx.DB) error {
    stmt := "TRUNCATE TABLE list, item;"

    if _, err := dbc.Exec(stmt); err != nil {
        return errors.Wrap(err, "truncate test database tables")
    }

    return nil
}
```

代码清单 3 展示了 `Truncate` 函数。顾名思义，它用于删除 `SeedLists` 和 `SeedItems` 函数插入的所有数据。

## 使用 testing.M 创建 TestMain

使用便于`填充/清除`数据库的软件包后，该集中精力配置以运行真正的集成测试了。Go 自带的测试工具可以让你在 `TestMain` 函数中定义需要的行为，在测试功能执行前执行。

### 代码清单 4

```golang
func TestMain(m *testing.M) {
    os.Exit(testMain(m))
}
```

<!-- markdown 代码行号问题 -->
代码清单 4 是 `TestMain` 函数，它在所有集成测试之前执行。在 23 行，叫做 `testMain` 的未导出的函数被 `os.Exit` 调用。这样做是为了 `testMain` 可以执行其中的延迟函数，并且仍可以在 `os.Exit` 调用内部设置适当的整数值。以下是 `testMain` 函数的实现。

### 代码清单 5

```golang
func testMain(m *testing.M) int {
    dbc, err := testdb.Open()
    if err != nil {
        log.WithError(err).Info("create test database connection")
        return 1
    }
    defer dbc.Close()

    a = handlers.NewApplication(dbc)

    return m.Run()
}
```

在代码清单 5 中，你可以看到 `testMain` 只有 8 行代码。28 行，函数调用 `testdb.Open()` 开始建立数据库连接。此调用的配置参数在 `testdb` 包中设置为常量。重要的是要注意，如果测试用的数据库未运行，调用 `Opne` 连接数据库会失败。该测试数据库是由 `docker-compose` 创建并提供的，详细说明在本系列的第 1 部分中（单击 [此处](https://www.ardanlabs.com/blog/2019/03/integration-testing-in-go-executing-tests-with-docker.html) 阅读第 1 部分）。

成功连接测试数据库后，连接将传递到 `handlers.NewApplication()`，并且此函数的返回值用于初始化 type 的包级变量 `*handlers.Application`。`handlers.Application` 类型对此项目是自定义的，并且具有用于 http.Handler 接口的 struct 字段，以简化 Web 服务的路由以及对已创建的开放数据库连接的引用。

现在，应用程序值已初始化，m.Run 可以调用它执行任何测试功能。对的调用 m.Run 处于阻塞状态，直到所有确定要运行的测试功能都执行完之后，该调用才会返回。非零退出代码表示失败，0 表示成功。

## 编写 Web 服务的集成测试

### 清单 6

```golang
func Test_getItems(t *testing.T) {
    defer func() {
        if err := testdb.Truncate(a.DB); err != nil {
            t.Errorf("error truncating test database tables: %v", err)
        }
    }()

    expectedLists, err := testdb.SeedLists(a.DB)
    if err != nil {
        t.Fatalf("error seeding lists: %v", err)
    }

    expectedItems, err := testdb.SeedItems(a.DB, expectedLists)
    if err != nil {
        t.Fatalf("error seeding items: %v", err)
    }
}
```

### 清单 7

```golang
// Application is the struct that contains the server handler as well as
// any references to services that the application needs.
type Application struct {
    DB      *sqlx.DB
    handler http.Handler
}

// ServeHTTP implements the http.Handler interface for the Application type.
func (a *Application) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    a.handler.ServeHTTP(w, r)
}

```

### 清单 8

```golang
req, err := http.NewRequest(http.MethodGet, fmt.Sprintf("/list/%d/item", test.ListID), nil)
if err != nil {
   t.Errorf("error creating request: %v", err)
}

w := httptest.NewRecorder()
a.ServeHTTP(w, req)
```

### 清单 9

```golang
if want, got := http.StatusOK, w.Code; want != got {
    t.Errorf("expected status code: %v, got status code: %v", want, got)
}
```

### 清单 10

```golang
var items []item.Item
resp := web.Response{
    Results: items,
}

if err := json.NewDecoder(w.Body).Decode(&resp); err != nil {
    t.Errorf("error decoding response body: %v", err)
}

if d := cmp.Diff(expectedItems, items); d != "" {
    t.Errorf("unexpected difference in response body:\n%v", d)
}
```

### 清单 11

```golang
// Add takes an indefinite amount of operands and adds them together, returning
// the sum of the operation.
func Add(operands ...int) int {
    var sum int

    for _, operand := range operands {
        sum += operand
    }

    return sum
}
```
### 清单 12

```golang
// TestAdd tests the Add function.
func TestAdd(t *testing.T) {
    tt := []struct {
        Name     string
        Operands []int
        Sum      int
    }{
        {
            Name:     "NoOperands",
            Operands: []int{},
            Sum:      0,
        },
        {
            Name:     "OneOperand",
            Operands: []int{10},
            Sum:      10,
        },
        {
            Name:     "TwoOperands",
            Operands: []int{10, 5},
            Sum:      15,
        },
        {
            Name:     "ThreeOperands",
            Operands: []int{10, 5, 4},
            Sum:      19,
        },
    }

    for _, test := range tt {
        fn := func(t *testing.T) {
            if e, a := test.Sum, Add(test.Operands...); e != a {
                t.Errorf("expected sum %d, got sum %d", e, a)
            }
        }

        t.Run(test.Name, fn)
    }
}
```

### 清单 13

```golang
// GenerateTempFile generates a temp file and returns the reference to
// the underlying os.File and an error.
func GenerateTempFile() (*os.File, error) {
    f, err := ioutil.TempFile("", "")
    if err != nil {
        return nil, err
    }

    return f, nil
}
```

### 清单 14

```golang
// GenerateTempFile generates a temp file and returns the reference to
// the underlying os.File.
func GenerateTempFile(t *testing.T) *os.File {
    t.Helper()

    f, err := ioutil.TempFile("", "")
    if err != nil {
        t.Fatalf("unable to generate temp file: %v", err)
    }

    return f
}
```

## 结论

如果不配置程序运行时所需的外部系统，则无法在集成测试的上下文中完全验证程序的行为。此外，需要持续监测那些外部系统（特别是当它们包含应用程序状态数据的情况下），以确保它们包含有效和有意义的数据。Go 使开发人员不仅可以在测试过程中进行配置，还可以无需标准库之外的包就能维护外部数据。因此，我们可以编写可读性，一致性，性能和可靠性同时都能保证的集成测试。Go 的真正魅力正在于其简约而功能齐全的工具集，它为开发人员提供了无需依赖外部库或任何非常规限制的功能。

---

via: <https://www.ardanlabs.com/blog/2019/10/integration-testing-in-go-set-up-and-writing-tests.html>

作者：[George Shaw](https://github.com/george-e-shaw-iv/)
译者：[TomatoAres](https://github.com/TomatoAres)
校对：[校对者 ID](https://github.com/校对者 ID)

本文由 [GCTT](https://github.com/studygolang/GCTT) 原创编译，[Go 中文网](https://studygolang.com/) 荣誉推出