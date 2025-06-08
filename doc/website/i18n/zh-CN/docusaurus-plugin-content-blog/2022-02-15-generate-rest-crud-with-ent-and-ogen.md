---
title: Auto generate REST CRUD with Ent and ogen 
author: MasseElch 
authorURL: "https://github.com/masseelch"
authorImageURL: "https://avatars.githubusercontent.com/u/12862103?v=4"
image: "https://entgo.io/images/assets/ogent/1.png"
---

2021年底，我们宣布[Ent](https://entgo.io)推出了一个全新官方扩展，用于生成完全兼容的[OpenAPI规范](https://swagger.io/resources/open-api/)文档：[`entoas`](https://github.com/ent/contrib/tree/master/entoas)。

今天，我们非常高兴地宣布，专为与`entoas`协同工作而构建的新扩展[`ogent`](https://github.com/ariga/ogent)正式发布。它利用[`ogen`](https://github.com/ogen-go/ogen)（[官网](https://ogen.dev/docs/intro/)）的强大能力，为`entoas`生成的OpenAPI规范文档提供类型安全且无需反射的实现方案。

`ogen`是一个基于OpenAPI规范v3文档的强约定型Go代码生成器，能同时生成服务端和客户端实现。开发者只需实现数据层访问接口即可。`ogen`具备诸多亮点功能，例如与[OpenTelemetry](https://opentelemetry.io/)的集成，强烈推荐您体验并给予支持。

本文介绍的扩展在Ent与[`ogen`](https://github.com/ogen-go/ogen)生成的代码之间架起桥梁，通过`entoas`的配置来补全`ogen`代码的缺失部分。

下图展示了Ent如何与`entoas`和`ogent`扩展协同工作，以及`ogen`在其中的作用。

<div style={{textAlign: 'center'}}>
  <img alt="Diagram" src="https://entgo.io/images/assets/ogent/1.png" />
  <p style={{fontSize: 12}}>Diagram</p>
</div>

若您是Ent新手，想了解如何连接各类数据库、执行迁移或操作实体，请参阅[入门教程](https://entgo.io/docs/tutorial-setup/)。

本文涉及的完整代码可在模块[示例库](https://github.com/ariga/ogent/tree/main/example/todo)中获取。

### 快速开始

:::note[]
虽然Ent支持Go 1.16+版本，但`ogen`要求至少使用1.17版本。
:::

要使用`ogent`扩展，请按[文档说明](https://entgo.io/docs/code-gen#use-entc-as-a-package)通过`entc`包进行代码生成。首先在Go模块中安装`entoas`和`ogent`扩展：

```shell
go get ariga.io/ogent@main
```

接下来按以下两步启用扩展并配置Ent：

1\. 创建名为`ent/entc.go`的Go文件并写入以下内容：

```go title="ent/entc.go"
//go:build ignore

package main

import (
	"log"

	"ariga.io/ogent"
	"entgo.io/contrib/entoas"
	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"github.com/ogen-go/ogen"
)

func main() {
	spec := new(ogen.Spec)
	oas, err := entoas.NewExtension(entoas.Spec(spec))
	if err != nil {
		log.Fatalf("creating entoas extension: %v", err)
	}
	ogent, err := ogent.NewExtension(spec)
	if err != nil {
		log.Fatalf("creating ogent extension: %v", err)
	}
	err = entc.Generate("./schema", &gen.Config{}, entc.Extensions(ogent, oas))
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

2\. 修改`ent/generate.go`文件以执行`ent/entc.go`：

```go title="ent/generate.go"
package ent

//go:generate go run -mod=mod entc.go
```

完成这些步骤后，即可基于您的Schema生成OAS文档并实现服务端代码！

### 生成CRUD HTTP API服务

构建HTTP API服务的第一步是创建Ent Schema图。以下是一个简明的示例Schema：

```go title="ent/schema/todo.go"
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
		field.Bool("done"),
	}
}
```

上述代码展示了Ent定义Schema图的典型方式，本例中我们创建了一个待办事项实体。

现在运行代码生成器：

```shell
go generate ./...
```

您将看到Ent代码生成器输出的大量文件。其中`ent/openapi.json`由`entoas`扩展生成，以下是其内容概览：

```json title="ent/openapi.json"
{
  "info": {
    "title": "Ent Schema API",
    "description": "This is an auto generated API description made out of an Ent schema definition",
    "termsOfService": "",
    "contact": {},
    "license": {
      "name": ""
    },
    "version": "0.0.0"
  },
  "paths": {
    "/todos": {
      "get": {
    [...]
```

<div style={{textAlign: 'center'}}>
  <img alt="Swagger Editor Example" src="https://entgo.io/images/assets/ogent/2.png" />
  <p style={{fontSize: 12}}>Swagger Editor Example</p>
</div>

不过，本文重点在于服务端实现部分，因此我们更关注名为`ent/ogent`的目录。所有以`_gen.go`结尾的文件均由`ogen`生成，其中`oas_server_gen.go`文件包含了`ogen`使用者需要实现的接口以运行服务端。

```go title="ent/ogent/oas_server_gen.go"
// Handler handles operations described by OpenAPI v3 specification.
type Handler interface {
	// CreateTodo implements createTodo operation.
	//
	// Creates a new Todo and persists it to storage.
	//
	// POST /todos
	CreateTodo(ctx context.Context, req CreateTodoReq) (CreateTodoRes, error)
	// DeleteTodo implements deleteTodo operation.
	//
	// Deletes the Todo with the requested ID.
	//
	// DELETE /todos/{id}
	DeleteTodo(ctx context.Context, params DeleteTodoParams) (DeleteTodoRes, error)
	// ListTodo implements listTodo operation.
	//
	// List Todos.
	//
	// GET /todos
	ListTodo(ctx context.Context, params ListTodoParams) (ListTodoRes, error)
	// ReadTodo implements readTodo operation.
	//
	// Finds the Todo with the requested ID and returns it.
	//
	// GET /todos/{id}
	ReadTodo(ctx context.Context, params ReadTodoParams) (ReadTodoRes, error)
	// UpdateTodo implements updateTodo operation.
	//
	// Updates a Todo and persists changes to storage.
	//
	// PATCH /todos/{id}
	UpdateTodo(ctx context.Context, req UpdateTodoReq, params UpdateTodoParams) (UpdateTodoRes, error)
}
```

`ogent`在`ogent.go`文件中为该处理器添加了实现。关于如何定义生成的路由及预加载边关系的配置，请参阅`entoas`的[文档](https://github.com/ent/contrib/entoas)。

以下展示一个生成的READ路由示例：

```go
// ReadTodo handles GET /todos/{id} requests.
func (h *OgentHandler) ReadTodo(ctx context.Context, params ReadTodoParams) (ReadTodoRes, error) {
	q := h.client.Todo.Query().Where(todo.IDEQ(params.ID))
	e, err := q.Only(ctx)
	if err != nil {
		switch {
		case ent.IsNotFound(err):
			return &R404{
				Code:   http.StatusNotFound,
				Status: http.StatusText(http.StatusNotFound),
				Errors: rawError(err),
			}, nil
		case ent.IsNotSingular(err):
			return &R409{
				Code:   http.StatusConflict,
				Status: http.StatusText(http.StatusConflict),
				Errors: rawError(err),
			}, nil
		default:
			// Let the server handle the error.
			return nil, err
		}
	}
	return NewTodoRead(e), nil
}
```

### 运行服务端

下一步是创建`main.go`文件并完成所有组件装配，构建一个用于提供Todo-API的应用服务端。下面的main函数初始化了SQLite内存数据库，执行迁移创建所需表结构，并按照`ent/openapi.json`文件的描述在`localhost:8080`上提供API服务：

```go title="main.go"
package main

import (
	"context"
	"log"
	"net/http"

	"entgo.io/ent/dialect"
	"entgo.io/ent/dialect/sql/schema"
	"<your-project>/ent"
	"<your-project>/ent/ogent"
	_ "github.com/mattn/go-sqlite3"
)

func main() {
	// Create ent client.
	client, err := ent.Open(dialect.SQLite, "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		log.Fatal(err)
	}
	// Run the migrations.
	if err := client.Schema.Create(context.Background(), schema.WithAtlas(true)); err != nil {
		log.Fatal(err)
	}
	// Start listening.
	srv, err := ogent.NewServer(ogent.NewOgentHandler(client))
	if err != nil {
		log.Fatal(err)
	}
	if err := http.ListenAndServe(":8080", srv); err != nil {
		log.Fatal(err)
	}
}
```

通过`go run -mod=mod main.go`启动服务后，即可开始使用API。

首先创建一个新Todo。为演示起见，我们首次不发送请求体：

```shell
↪ curl -X POST -H "Content-Type: application/json" localhost:8080/todos
{
  "error_message": "body required"
}
```

可见`ogen`已自动处理该情况，因为`entoas`在创建新资源时将请求体标记为必填项。现在带上请求体再次尝试：

```shell
↪ curl -X POST -H "Content-Type: application/json" -d '{"title":"Give ogen and ogent a Star on GitHub"}'  localhost:8080/todos
{
  "error_message": "decode CreateTodo:application/json request: invalid: done (field required)"
}
```

糟糕！问题出在哪里？`ogen`会给出提示：`done`字段是必填项。要修复这个问题，请修改schema定义将该字段设为可选：

```go {18} title="ent/schema/todo.go"
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
		field.Bool("done").
		    Optional(),
	}
}
```

由于配置发生了变更，需要重新执行代码生成并重启服务端：

```shell
go generate ./...
go run -mod=mod main.go
```

现在再次尝试创建Todo，观察结果：

```shell
↪ curl -X POST -H "Content-Type: application/json" -d '{"title":"Give ogen and ogent a Star on GitHub"}'  localhost:8080/todos
{
  "id": 1,
  "title": "Give ogen and ogent a Star on GitHub",
  "done": false
}
```

成功！数据库中已新增一条Todo记录！

假设您已完成Todo事项并为[`ogen`](https://github.com/ogen-go/ogen)和[`ogent`](https://github.com/ariga/ogent)点了赞（**强烈推荐！**），可通过PATCH请求标记Todo为已完成状态：

```shell
↪ curl -X PATCH -H "Content-Type: application/json" -d '{"done":true}'  localhost:8080/todos/1
{
  "id": 1,
  "title": "Give ogen and ogent a Star on GitHub",
  "done": true
}
```

### 添加自定义端点

虽然当前已能标记Todo完成状态，但若增加专属路由`PATCH todos/:id/done`会更优雅。实现此功能需要完成两件事：在OAS文档中描述新路由，并实现该路由。首先通过`entoas`的mutation builder添加路由描述，编辑`ent/entc.go`文件：

```go {17-37} title="ent/entc.go"
//go:build ignore

package main

import (
	"log"

	"entgo.io/contrib/entoas"
	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"github.com/ariga/ogent"
	"github.com/ogen-go/ogen"
)

func main() {
	spec := new(ogen.Spec)
	oas, err := entoas.NewExtension(
		entoas.Spec(spec),
		entoas.Mutations(func(_ *gen.Graph, spec *ogen.Spec) error {
			spec.AddPathItem("/todos/{id}/done", ogen.NewPathItem().
				SetDescription("Mark an item as done").
				SetPatch(ogen.NewOperation().
					SetOperationID("markDone").
					SetSummary("Marks a todo item as done.").
					AddTags("Todo").
					AddResponse("204", ogen.NewResponse().SetDescription("Item marked as done")),
				).
				AddParameters(ogen.NewParameter().
					InPath().
					SetName("id").
					SetRequired(true).
					SetSchema(ogen.Int()),
				),
			)
			return nil
		}),
	)
	if err != nil {
		log.Fatalf("creating entoas extension: %v", err)
	}
	ogent, err := ogent.NewExtension(spec)
	if err != nil {
		log.Fatalf("creating ogent extension: %v", err)
	}
	err = entc.Generate("./schema", &gen.Config{}, entc.Extensions(ogent, oas))
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

执行代码生成命令(`go generate ./...`)后，`ent/openapi.json`文件中将出现新条目：

```json
"/todos/{id}/done": {
  "description": "Mark an item as done",
  "patch": {
    "tags": [
      "Todo"    
    ],
    "summary": "Marks a todo item as done.",
    "operationId": "markDone",
    "responses": {
      "204": {
        "description": "Item marked as done"
      }
    }
  },
  "parameters": [
    {
      "name": "id",
      "in": "path",
      "schema": {
        "type": "integer"
      },
      "required": true
    }
  ]
}
```

<div style={{textAlign: 'center'}}>
  <img alt="Custom Endpoint" src="https://entgo.io/images/assets/ogent/3.png" />
  <p style={{fontSize: 12}}>Custom Endpoint</p>
</div>

由`ogen`生成的`ent/ogent/oas_server_gen.go`文件也会同步更新：

```go {21-24} title="ent/ogent/oas_server_gen.go"
// Handler handles operations described by OpenAPI v3 specification.
type Handler interface {
	// CreateTodo implements createTodo operation.
	//
	// Creates a new Todo and persists it to storage.
	//
	// POST /todos
	CreateTodo(ctx context.Context, req CreateTodoReq) (CreateTodoRes, error)
	// DeleteTodo implements deleteTodo operation.
	//
	// Deletes the Todo with the requested ID.
	//
	// DELETE /todos/{id}
	DeleteTodo(ctx context.Context, params DeleteTodoParams) (DeleteTodoRes, error)
	// ListTodo implements listTodo operation.
	//
	// List Todos.
	//
	// GET /todos
	ListTodo(ctx context.Context, params ListTodoParams) (ListTodoRes, error)
	// MarkDone implements markDone operation.
	//
	// PATCH /todos/{id}/done
	MarkDone(ctx context.Context, params MarkDoneParams) (MarkDoneNoContent, error)
	// ReadTodo implements readTodo operation.
	//
	// Finds the Todo with the requested ID and returns it.
	//
	// GET /todos/{id}
	ReadTodo(ctx context.Context, params ReadTodoParams) (ReadTodoRes, error)
	// UpdateTodo implements updateTodo operation.
	//
	// Updates a Todo and persists changes to storage.
	//
	// PATCH /todos/{id}
	UpdateTodo(ctx context.Context, req UpdateTodoReq, params UpdateTodoParams) (UpdateTodoRes, error)
}
```

若此时运行服务端，Go编译器会报错，因为`ogent`代码生成器尚未实现新路由。需手动完成实现。替换当前`main.go`文件内容以添加新方法：

```go {15-22,34-38,40} title="main.go"
package main

import (
	"context"
	"log"
	"net/http"

	"entgo.io/ent/dialect"
	"entgo.io/ent/dialect/sql/schema"
	"github.com/ariga/ogent/example/todo/ent"
	"github.com/ariga/ogent/example/todo/ent/ogent"
	_ "github.com/mattn/go-sqlite3"
)

type handler struct {
	*ogent.OgentHandler
	client *ent.Client
}

func (h handler) MarkDone(ctx context.Context, params ogent.MarkDoneParams) (ogent.MarkDoneNoContent, error) {
	return ogent.MarkDoneNoContent{}, h.client.Todo.UpdateOneID(params.ID).SetDone(true).Exec(ctx)
}

func main() {
	// Create ent client.
	client, err := ent.Open(dialect.SQLite, "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		log.Fatal(err)
	}
	// Run the migrations.
	if err := client.Schema.Create(context.Background(), schema.WithAtlas(true)); err != nil {
		log.Fatal(err)
	}
	// Create the handler.
	h := handler{
		OgentHandler: ogent.NewOgentHandler(client),
		client:       client,
	}
	// Start listening.
	srv := ogent.NewServer(h)
	if err := http.ListenAndServe(":8180", srv); err != nil {
		log.Fatal(err)
	}
}

```

重启服务后，即可通过以下请求标记Todo事项为已完成状态：

```shell
↪ curl -X PATCH localhost:8180/todos/1/done
```

### 未来计划

`ogent` 还有一些改进计划，最值得注意的是为 LIST 路由添加代码生成、类型安全的过滤功能。我们希望能先听取您的反馈意见。

### 总结

本文我们发布了 `ogent`，这是针对 `entoas` 生成的 OpenAPI 规范文档的官方实现生成器。该扩展利用了 [`ogen`](https://github.com/ogen-go/ogen) 的强大功能——一个功能丰富且强大的 OpenAPI v3 文档 Go 代码生成器，来提供开箱即用、可扩展的 RESTful HTTP API 服务器。

请注意，`ogen` 和 `entoas`/`ogent` 尚未发布首个主要版本，仍在开发中。不过，API 可以认为是稳定的。

有问题吗？需要入门帮助？欢迎加入我们的 [Discord 服务器](https://discord.gg/qZmPgTE6RX) 或 [Slack 频道](https://entgo.io/docs/slack/)。

:::note[获取更多 Ent 新闻和更新：]

- 订阅我们的 [Newsletter](https://entgo.substack.com/)
- 在 [Twitter](https://twitter.com/entgo_io) 上关注我们
- 加入 [Gophers Slack](https://entgo.io/docs/slack) 的 #ent 频道
- 加入 [Ent Discord 服务器](https://discord.gg/qZmPgTE6RX)

:::