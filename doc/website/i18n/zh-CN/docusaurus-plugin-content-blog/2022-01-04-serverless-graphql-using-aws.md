---
title: Serverless GraphQL using with AWS and ent
author: Bodo Kaiser
authorURL: "https://github.com/bodokaiser"
authorImageURL: "https://avatars.githubusercontent.com/u/1780466?v=4"
image: https://entgo.io/images/assets/appsync/share.png
---

[GraphQL][1] 是一种面向HTTP API的查询语言，通过静态类型接口便捷地呈现当今复杂的数据层级结构。使用GraphQL的一种方式是导入实现GraphQL服务器的库，并注册自定义解析器来实现数据库接口。另一种方式是使用GraphQL云服务来实现GraphQL服务器，并将无服务器云函数注册为解析器。云服务的众多优势中，最显著的实践价值在于解析器的独立性和可组合性——例如我们可以分别为关系型数据库和搜索数据库编写不同的解析器。

下文我们将基于[亚马逊云服务(AWS)][2]搭建此类架构，具体采用[AWS AppSync][3]作为GraphQL云服务，使用[AWS Lambda][4]运行关系型数据库解析器，该解析器通过[Go][5]语言开发并采用[Ent][6]作为实体框架。相比AWS Lambda最流行的Nodejs运行时，Go具备更快的启动速度、更高的运行时性能，以及（个人认为）更优的开发者体验。而Ent作为补充，提供了类型安全访问关系型数据库的创新方案，在我看来这是Go生态中无可匹敌的。综上所述，将Ent与AWS Lambda结合作为AWS AppSync解析器，能极佳地应对当今严苛的API需求。

后续章节我们将依次完成：在AWS AppSync中配置GraphQL、部署运行Ent的AWS Lambda函数、提出集成Ent与AWS Lambda事件处理器的Go实现方案、快速测试Ent函数，最终将其注册为AppSync API的数据源并配置解析器（用于定义GraphQL请求到AWS Lambda事件的映射）。请注意本教程需要AWS账户和**可公开访问的Postgres数据库URL**，可能会产生费用。

### 配置AWS AppSync架构

登录AWS账户后，通过导航栏选择AppSync服务。在服务落地页点击"Create API"按钮，进入"Getting Started"页面：

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of getting started with AWS AppSync from scratch" src="https://entgo.io/images/assets/appsync/from-scratch.png" />
  <p style={{fontSize: 12}}>Getting started from scratch with AWS AppSync</p>
</div>

在顶部标有"Customize your API or import from Amazon DynamoDB"的面板中选择"Build from scratch"选项，点击该面板的"Start"按钮。此时会出现API命名表单，本教程输入"Todo"（如下图所示），点击"Create"按钮确认。

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of creating a new AWS AppSync API resource" src="https://entgo.io/images/assets/appsync/create-resources.png" />
  <p style={{fontSize: 12}}>Creating a new API resource in AWS AppSync</p>
</div>

创建完成后，落地页将显示三个面板：架构定义面板、API查询面板、以及AppSync应用集成面板（如下图所示）。

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of the landing page of the AWS AppSync API" src="https://entgo.io/images/assets/appsync/getting-started.png" />
  <p style={{fontSize: 12}}>Landing page of the AWS AppSync API</p>
</div>

点击第一个面板的"Edit Schema"按钮，用以下GraphQL架构替换原有内容：

```graphql
input AddTodoInput {
	title: String!
}

type AddTodoOutput {
	todo: Todo!
}

type Mutation {
	addTodo(input: AddTodoInput!): AddTodoOutput!
	removeTodo(input: RemoveTodoInput!): RemoveTodoOutput!
}

type Query {
	todos: [Todo!]!
	todo(id: ID!): Todo
}

input RemoveTodoInput {
	todoId: ID!
}

type RemoveTodoOutput {
	todo: Todo!
}

type Todo {
	id: ID!
	title: String!
}

schema {
	query: Query
	mutation: Mutation
}
```

替换完成后会进行简短验证，点击右上角"Save Schema"按钮保存，此时界面应如下图所示：

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot AWS AppSync: Final GraphQL schema for AWS AppSync API" src="https://entgo.io/images/assets/appsync/final-schema.png" />
  <p style={{fontSize: 12}}>Final GraphQL schema of AWS AppSync API</p>
</div>

若此时向AppSync API发送GraphQL请求，由于尚未绑定解析器，API将返回错误。我们将在通过AWS Lambda部署Ent函数后配置解析器。

本文不对GraphQL架构展开详细说明。简而言之，该架构通过`Query.todos`实现待办列表查询，通过`Query.todo`实现单条查询，通过`Mutation.createTodo`实现创建，通过`Mutation.deleteTodo`实现删除。这个GraphQL API类似于简单的REST API设计：使用`GET /todos`、`GET /todos/:id`、`POST /todos`和`DELETE /todos/:id`来操作待办资源。关于GraphQL架构设计的细节（如`Query`和`Mutation`对象的参数与返回值），我遵循[GitHub GraphQL API](https://docs.github.com/en/graphql/reference/queries)的实践方案。

### 配置AWS Lambda函数

在完成AppSync API配置后，下一步是部署运行Ent的AWS Lambda函数。我们通过导航栏进入AWS Lambda服务，页面将显示函数列表：

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of AWS Lambda landing page listing functions" src="https://entgo.io/images/assets/appsync/function-list.png" />
  <p style={{fontSize: 12}}>AWS Lambda landing page showing functions.</p>
</div>

点击右上角"创建函数"按钮，在上方面板选择"从头开始编写"。将函数命名为"ent"，运行时选择"Go 1.x"，然后点击底部"创建函数"按钮。随后将进入该函数的详情页面：

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of AWS Lambda landing page listing functions" src="https://entgo.io/images/assets/appsync/function-overview.png" />
  <p style={{fontSize: 12}}>AWS Lambda function overview of the Ent function.</p>
</div>

在审查Go代码和上传编译好的二进制文件前，我们需要调整"ent"函数的默认设置。首先将默认处理程序名称从`hello`改为`main`（对应Go编译后的二进制文件名）：

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of AWS Lambda landing page listing functions" src="https://entgo.io/images/assets/appsync/runtime-settings.png" />
  <p style={{fontSize: 12}}>AWS Lambda runtime settings of Ent function.</p>
</div>

其次添加环境变量`DATABASE_URL`，用于存储数据库网络参数和凭据：

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of AWS Lambda landing page listing functions" src="https://entgo.io/images/assets/appsync/envars.png" />
  <p style={{fontSize: 12}}>AWS Lambda environment variables settings of Ent function.</p>
</div>

连接数据库时需传入[数据源名称(DSN)](https://en.wikipedia.org/wiki/Data_source_name)，例如`postgres://用户名:密码@主机名/数据库名`。AWS Lambda默认会加密环境变量，这是传递数据库连接参数既快速又安全的方式。替代方案包括使用AWS密钥管理服务动态获取凭证（支持凭证轮换等），或通过AWS IAM处理数据库授权。

若Postgres数据库创建于AWS RDS，默认用户名和数据库名均为`postgres`。可通过修改RDS实例来重置密码。

### 配置Ent并部署AWS Lambda

现在我们将审查、编译数据库Go二进制文件并部署到"ent"函数。完整源代码参见[bodokaiser/entgo-aws-appsync](https://github.com/bodokaiser/entgo-aws-appsync)。

首先创建并进入空目录：

```console
mkdir entgo-aws-appsync
cd entgo-aws-appsync
```

接着初始化新的Go模块：

```console
go mod init entgo-aws-appsync
```

然后创建`Todo`模型并引入ent依赖：

```console
go run -mod=mod entgo.io/ent/cmd/ent new Todo
```

并添加`title`字段：

```go {15-17} title="ent/schema/todo.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

// Todo holds the schema definition for the Todo entity.
type Todo struct {
	ent.Schema
}

// Fields of the Todo.
func (Todo) Fields() []ent.Field {
	return []ent.Field{
		field.String("title"),
	}
}

// Edges of the Todo.
func (Todo) Edges() []ent.Edge {
	return nil
}
```

最后执行Ent代码生成：

```console
go generate ./ent
```

我们使用Ent编写了实现todo增删查操作的解析器函数集：

```go title="internal/handler/resolver.go"
package resolver

import (
	"context"
	"fmt"
	"strconv"

	"entgo-aws-appsync/ent"
	"entgo-aws-appsync/ent/todo"
)

// TodosInput is the input to the Todos query.
type TodosInput struct{}

// Todos queries all todos.
func Todos(ctx context.Context, client *ent.Client, input TodosInput) ([]*ent.Todo, error) {
	return client.Todo.
		Query().
		All(ctx)
}

// TodoByIDInput is the input to the TodoByID query.
type TodoByIDInput struct {
	ID string `json:"id"`
}

// TodoByID queries a single todo by its id.
func TodoByID(ctx context.Context, client *ent.Client, input TodoByIDInput) (*ent.Todo, error) {
	tid, err := strconv.Atoi(input.ID)
	if err != nil {
		return nil, fmt.Errorf("failed parsing todo id: %w", err)
	}
	return client.Todo.
		Query().
		Where(todo.ID(tid)).
		Only(ctx)
}

// AddTodoInput is the input to the AddTodo mutation.
type AddTodoInput struct {
	Title string `json:"title"`
}

// AddTodoOutput is the output to the AddTodo mutation.
type AddTodoOutput struct {
	Todo *ent.Todo `json:"todo"`
}

// AddTodo adds a todo and returns it.
func AddTodo(ctx context.Context, client *ent.Client, input AddTodoInput) (*AddTodoOutput, error) {
	t, err := client.Todo.
		Create().
		SetTitle(input.Title).
		Save(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed creating todo: %w", err)
	}
	return &AddTodoOutput{Todo: t}, nil
}

// RemoveTodoInput is the input to the RemoveTodo mutation.
type RemoveTodoInput struct {
	TodoID string `json:"todoId"`
}

// RemoveTodoOutput is the output to the RemoveTodo mutation.
type RemoveTodoOutput struct {
	Todo *ent.Todo `json:"todo"`
}

// RemoveTodo removes a todo and returns it.
func RemoveTodo(ctx context.Context, client *ent.Client, input RemoveTodoInput) (*RemoveTodoOutput, error) {
	t, err := TodoByID(ctx, client, TodoByIDInput{ID: input.TodoID})
	if err != nil {
		return nil, fmt.Errorf("failed querying todo with id %q: %w", input.TodoID, err)
	}
	err = client.Todo.
		DeleteOne(t).
		Exec(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed deleting todo with id %q: %w", input.TodoID, err)
	}
	return &RemoveTodoOutput{Todo: t}, nil
}
```

解析器函数采用输入结构体映射GraphQL请求参数，输出结构体可返回多个对象以支持复杂操作。

为实现Lambda事件到解析函数的映射，我们编写了根据事件中`action`字段进行路由的处理器：

```go title="internal/handler/handler.go"
package handler

import (
	"context"
	"encoding/json"
	"fmt"
	"log"

	"entgo-aws-appsync/ent"
	"entgo-aws-appsync/internal/resolver"
)

// Action specifies the event type.
type Action string

// List of supported event actions.
const (
	ActionMigrate Action = "migrate"

	ActionTodos      = "todos"
	ActionTodoByID   = "todoById"
	ActionAddTodo    = "addTodo"
	ActionRemoveTodo = "removeTodo"
)

// Event is the argument of the event handler.
type Event struct {
	Action Action          `json:"action"`
	Input  json.RawMessage `json:"input"`
}

// Handler handles supported events.
type Handler struct {
	client *ent.Client
}

// Returns a new event handler.
func New(c *ent.Client) *Handler {
	return &Handler{
		client: c,
	}
}

// Handle implements the event handling by action.
func (h *Handler) Handle(ctx context.Context, e Event) (interface{}, error) {
	log.Printf("action %s with payload %s\n", e.Action, e.Input)

	switch e.Action {
	case ActionMigrate:
		return nil, h.client.Schema.Create(ctx)
	case ActionTodos:
		var input resolver.TodosInput
		return resolver.Todos(ctx, h.client, input)
	case ActionTodoByID:
		var input resolver.TodoByIDInput
		if err := json.Unmarshal(e.Input, &input); err != nil {
			return nil, fmt.Errorf("failed parsing %s params: %w", ActionTodoByID, err)
		}
		return resolver.TodoByID(ctx, h.client, input)
	case ActionAddTodo:
		var input resolver.AddTodoInput
		if err := json.Unmarshal(e.Input, &input); err != nil {
			return nil, fmt.Errorf("failed parsing %s params: %w", ActionAddTodo, err)
		}
		return resolver.AddTodo(ctx, h.client, input)
	case ActionRemoveTodo:
		var input resolver.RemoveTodoInput
		if err := json.Unmarshal(e.Input, &input); err != nil {
			return nil, fmt.Errorf("failed parsing %s params: %w", ActionRemoveTodo, err)
		}
		return resolver.RemoveTodo(ctx, h.client, input)
	}

	return nil, fmt.Errorf("invalid action %q", e.Action)
}
```

除解析操作外，我们还添加了迁移操作作为暴露数据库迁移的便捷方式。

最后需要将`Handler`实例注册到AWS Lambda库：

```go title="lambda/main.go"
package main

import (
	"database/sql"
	"log"
	"os"

	"entgo.io/ent/dialect"
	entsql "entgo.io/ent/dialect/sql"

	"github.com/aws/aws-lambda-go/lambda"
	_ "github.com/jackc/pgx/v4/stdlib"

	"entgo-aws-appsync/ent"
	"entgo-aws-appsync/internal/handler"
)

func main() {
	// open the database connection using the pgx driver
	db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatalf("failed opening database connection: %v", err)
	}

	// initiate the ent database client for the Postgres database
	client := ent.NewClient(ent.Driver(entsql.OpenDB(dialect.Postgres, db)))
	defer client.Close()

	// register our event handler to listen on Lambda events
	lambda.Start(handler.New(client).Handle)
}
```

`main`的函数体会在Lambda冷启动时执行。冷启动后函数进入"热"状态，仅执行事件处理器代码，这使得Lambda执行非常高效。

运行以下命令编译并部署Go代码：

```console
GOOS=linux go build -o main ./lambda
zip function.zip main
aws lambda update-function-code --function-name ent --zip-file fileb://function.zip
```

第一条命令会生成名为`main`的编译后二进制文件。第二条命令将二进制文件压缩为ZIP归档，这是AWS Lambda所要求的格式。第三条命令用新的ZIP归档替换名为`ent`的AWS Lambda函数代码。若需操作多个AWS账户，可使用`--profile <your aws profile>`参数切换配置。

成功部署AWS Lambda后，在Web控制台中打开"ent"函数的"Test"标签页，通过"migrate"操作触发测试：

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of invoking the Ent Lambda with a migrate action" src="https://entgo.io/images/assets/appsync/execution-result.png" />
  <p style={{fontSize: 12}}>Invoking Lambda with a "migrate" action</p>
</div>

若成功执行，您将看到绿色反馈框，此时可测试"todos"操作的结果：

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of invoking the Ent Lambda with a todos action" src="https://entgo.io/images/assets/appsync/execution-result2.png" />
  <p style={{fontSize: 12}}>Invoking Lambda with a "todos" action</p>
</div>

若测试执行失败，很可能是数据库连接存在问题。

### 配置AWS AppSync解析器

成功部署"ent"函数后，我们需要将其注册为AppSync API的数据源，并配置模式解析器以将AppSync请求映射到Lambda事件。首先在Web控制台中打开AWS AppSync API，左侧导航面板选择"Data Sources"。

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of the list of data sources registered to the AWS AppSync API" src="https://entgo.io/images/assets/appsync/data-sources.png" />
  <p style={{fontSize: 12}}>List of data sources registered to the AWS AppSync API</p>
</div>

点击右上角的"Create data source"按钮，开始将"ent"函数注册为数据源：

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot registering the ent Lambda as data source to the AWS AppSync API" src="https://entgo.io/images/assets/appsync/new-data-source.png" />
  <p style={{fontSize: 12}}>Registering the ent Lambda as data source to the AWS AppSync API</p>
</div>

现在打开AppSync API的GraphQL模式，在右侧边栏中找到`Query`类型。点击`Query.Todos`类型旁的"Attach"按钮：

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot attaching a resolver to Query type in the AWS AppSync API" src="https://entgo.io/images/assets/appsync/todo-schema.png" />
  <p style={{fontSize: 12}}>Attaching a resolver for the todos Query in the AWS AppSync API</p>
</div>

在`Query.todos`的解析器视图中，选择Lambda函数作为数据源，启用请求映射模板选项，

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot configuring the resolver mapping for the todos Query in the AWS AppSync API" src="https://entgo.io/images/assets/appsync/edit-resolver.png" />
  <p style={{fontSize: 12}}>Configuring the resolver mapping for the todos Query in the AWS AppSync API</p>
</div>

并复制以下模板：

```vtl title="Query.todos"
{
  "version" : "2017-02-28",
  "operation": "Invoke",
  "payload": {
    "action": "todos"
  }
}
```

对剩余的`Query`和`Mutation`类型重复相同流程：

```vtl title="Query.todo"
{
  "version" : "2017-02-28",
  "operation": "Invoke",
  "payload": {
    "action": "todo",
    "input": $util.toJson($context.args.input)
  }
}
```

```vtl title="Mutation.addTodo"
{
  "version" : "2017-02-28",
  "operation": "Invoke",
  "payload": {
    "action": "addTodo",
    "input": $util.toJson($context.args.input)
  }
}
```

```vtl title="Mutation.removeTodo"
{
  "version" : "2017-02-28",
  "operation": "Invoke",
  "payload": {
    "action": "removeTodo",
    "input": $util.toJson($context.args.input)
  }
}
```

请求映射模板让我们可以构造调用Lambda函数时使用的事件对象。通过`$context`对象，我们能访问GraphQL请求和认证会话。此外，还可以顺序排列多个解析器，并通过`$context`对象引用各自的输出。原则上也可以定义响应映射模板，但在大多数情况下直接"原样"返回响应对象就已足够。

### 使用Query Explorer测试AppSync

最简单的测试方式是使用AWS AppSync中的Query Explorer。另一种方法是在AppSync API设置中注册API密钥，然后使用任何标准GraphQL客户端。

首先创建标题为`foo`的待办事项：

```graphql
mutation MyMutation {
  addTodo(input: {title: "foo"}) {
    todo {
      id
      title
    }
  }
}
```

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of an executed addTodo Mutation using the AppSync Query Explorer" src="https://entgo.io/images/assets/appsync/todo-queries.png" />
  <p style={{fontSize: 12}}>"addTodo" Mutation using the AppSync Query Explorer</p>
</div>

请求待办事项列表时应返回单个标题为`foo`的条目：

```graphql
query MyQuery {
  todos {
    title
    id
  }
}
```

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of an executed addTodo Mutation using the AppSync Query Explorer" src="https://entgo.io/images/assets/appsync/todo-queries-3.png" />
  <p style={{fontSize: 12}}>"addTodo" Mutation using the AppSync Query Explorer</p>
</div>

通过ID查询`foo`待办事项也应成功：

```graphql
query MyQuery {
  todo(id: "1") {
    title
    id
  }
}
```

<div style={{textAlign: 'center'}}>
  <img alt="Screenshot of an executed addTodo Mutation using the AppSync Query Explorer" src="https://entgo.io/images/assets/appsync/todo-queries-4.png" />
  <p style={{fontSize: 12}}>"addTodo" Mutation using the AppSync Query Explorer</p>
</div>

### 总结

我们成功部署了用于管理简单待办事项的无服务器GraphQL API，使用AWS AppSync、AWS Lambda和Ent实现。特别提供了通过Web控制台配置AWS AppSync和AWS Lambda的逐步指南，同时还讨论了Go代码结构的设计方案。

本文未涉及在AWS中测试和搭建数据库基础设施的内容。在无服务器架构中，这些方面比传统模式更具挑战性。例如，当多个Lambda函数并行冷启动时，数据库连接池会迅速耗尽，此时需要数据库代理。此外，由于无法轻松隔离运行云服务，我们只能进行本地测试和端到端测试，这要求我们重新思考测试策略。

尽管如此，所提出的GraphQL服务器方案能很好地扩展到现实应用的复杂需求中，既能受益于无服务器基础设施的优势，又能享受Ent框架带来的愉悦开发体验。

有问题需要帮助？欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack/)。

:::note[获取更多Ent资讯：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter账号](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::

[1]: https://graphql.org

[2]: https://aws.amazon.com

[3]: https://aws.amazon.com/appsync/

[4]: https://aws.amazon.com/lambda/

[5]: https://go.dev

[6]: https://entgo.io