---
id: graphql
title: GraphQL Integration
---

Ent框架通过[99designs/gqlgen](https://github.com/99designs/gqlgen)库提供GraphQL支持，主要集成功能包括：

1. 为Ent模式中定义的节点和边生成GraphQL模式
2. 自动生成`Query`和`Mutation`解析器，并与[Relay框架](https://relay.dev/)无缝集成
3. 支持过滤、分页（包括嵌套分页）及兼容[Relay游标连接规范](https://relay.dev/graphql/connections.htm)
4. 高效的[字段收集](tutorial-todo-gql-field-collection.md)机制，无需数据加载器即可解决N+1问题
5. [事务性变更](tutorial-todo-gql-tx-mutation.md)确保故障时的数据一致性

更多信息请参阅官网的[GraphQL教程](tutorial-todo-gql.mdx#basic-setup)

## 快速入门

启用[`entgql`](https://github.com/ent/contrib/tree/master/entgql)扩展需要按照[文档](code-gen.md#use-entc-as-a-package)使用`entc`包，具体分三步：

1\. 创建`ent/entc.go`文件并写入以下内容：

```go title="ent/entc.go"
// +build ignore

package main

import (
	"log"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"entgo.io/contrib/entgql"
)

func main() {
	ex, err := entgql.NewExtension()
	if err != nil {
		log.Fatalf("creating entgql extension: %v", err)
	}
	if err := entc.Generate("./schema", &gen.Config{}, entc.Extensions(ex)); err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

2\. 修改`ent/generate.go`文件来执行`ent/entc.go`：

```go title="ent/generate.go"
package ent

//go:generate go run -mod=mod entc.go
```

注意`ent/entc.go`通过构建标签被忽略，实际由`generate.go`在`go generate`时执行。完整示例参见[ent/contrib仓库](https://github.com/ent/contrib/blob/master/entgql/internal/todo)

3\. 为Ent项目运行代码生成：

```console
go generate ./...
```

代码生成完成后，项目将新增以下功能

## 节点API

新建的`ent/gql_node.go`文件实现了[Relay节点接口](https://relay.dev/graphql/objectidentification.htm)

要在[GraphQL解析器](https://gqlgen.com/reference/resolvers/)中使用生成的`ent.Noder`接口，需在查询解析器中添加`Node`方法，具体用法参考[配置章节](#gql-configuration)

若在模式迁移中使用[全局ID](migrate.md#universal-ids)选项，节点类型可从ID值推导：

```go
func (r *queryResolver) Node(ctx context.Context, id int) (ent.Noder, error) {
	return r.client.Noder(ctx, id)
}
```

若使用自定义全局ID格式，可如下控制节点类型：

```go
func (r *queryResolver) Node(ctx context.Context, guid string) (ent.Noder, error) {
	typ, id := parseGUID(guid)
	return r.client.Noder(ctx, id, ent.WithFixedNodeType(typ))
}
```

## GraphQL配置

以下是[todo应用示例](https://github.com/ent/contrib/tree/master/entgql/internal/todo)的配置范例：

```yaml
schema:
  - todo.graphql

resolver:
  # Tell gqlgen to generate resolvers next to the schema file.
  layout: follow-schema
  dir: .

# gqlgen will search for any type names in the schema in the generated
# ent package. If they match it will use them, otherwise it will new ones.
autobind:
  - entgo.io/contrib/entgql/internal/todo/ent

models:
  ID:
    model:
      - github.com/99designs/gqlgen/graphql.IntID
  Node:
    model:
      # ent.Noder is the new interface generated by the Node template.
      - entgo.io/contrib/entgql/internal/todo/ent.Noder
```

## 分页

分页模板根据_Relay游标连接规范_实现分页支持，规范详情见[官网](https://relay.dev/graphql/connections.htm)

## 连接排序

排序功能允许对连接返回的边进行排序

### 使用说明

- 生成的类型将自动绑定（`autobind`）到GraphQL类型（需保持命名规范，参见下方示例）。
- 排序字段通常应建立[索引](schema-indexes.md)以避免全表扫描。
- 分页查询仅支持按单个字段排序（不支持多级排序语义）。

### 示例

以下是为现有GraphQL类型添加排序功能的具体步骤。代码示例基于[ent/contrib/entql/todo](https://github.com/ent/contrib/tree/master/entgql/internal/todo)中的待办事项应用。

### 在ent/schema中定义排序字段

通过`entgql.Annotation`注解可在任意可比较的ent字段上定义排序。注意`OrderField`名称必须与GraphQL schema中的枚举值匹配。

```go
func (Todo) Fields() []ent.Field {
    return []ent.Field{
	    field.Time("created_at").
			Default(time.Now).
			Immutable().
			Annotations(
				entgql.OrderField("CREATED_AT"),
			),
		field.Enum("status").
			NamedValues(
				"InProgress", "IN_PROGRESS",
				"Completed", "COMPLETED",
			).
			Annotations(
				entgql.OrderField("STATUS"),
			),
		field.Int("priority").
			Default(0).
			Annotations(
				entgql.OrderField("PRIORITY"),
			),
		field.Text("text").
			NotEmpty().
			Annotations(
				entgql.OrderField("TEXT"),
			),
    }
}
```

完成schema修改后，请运行`go generate`使其生效。

### 在GraphQL schema中定义排序类型

接下来需在GraphQL schema中定义排序类型：

```graphql
enum OrderDirection {
  ASC
  DESC
}

enum TodoOrderField {
  CREATED_AT
  PRIORITY
  STATUS
  TEXT
}

input TodoOrder {
  direction: OrderDirection!
  field: TodoOrderField
}
```

注意命名必须遵循`<T>OrderField`/`<T>Order`格式以实现与生成ent类型的自动绑定。也可使用[@goModel](https://gqlgen.com/config/#inline-config-with-directives)指令手动绑定类型。

### 为分页查询添加orderBy参数

```graphql
type Query {
  todos(
    after: Cursor
    first: Int
    before: Cursor
    last: Int
    orderBy: TodoOrder
  ): TodoConnection!
}
```

完成GraphQL schema修改后，运行`gqlgen`代码生成。

### 更新底层解析器

在Todo解析器中更新代码，将`orderBy`参数传递给`.Paginate()`调用：

```go
func (r *queryResolver) Todos(ctx context.Context, after *ent.Cursor, first *int, before *ent.Cursor, last *int, orderBy *ent.TodoOrder) (*ent.TodoConnection, error) {
	return r.client.Todo.Query().
		Paginate(ctx, after, first, before, last,
			ent.WithTodoOrder(orderBy),
		)
}
```

### 在GraphQL中使用

```graphql
query {
    todos(first: 3, orderBy: {direction: DESC, field: TEXT}) {
        edges {
            node {
                text
            }
        }
    }
}
```

## 字段收集

字段收集模板通过预加载机制实现了ent边关系的自动[GraphQL字段收集](https://spec.graphql.org/June2018/#sec-Field-Collection)。当查询请求节点及其边关系时，entgql会自动为根查询添加[`With<E>`](eager-load.mdx#api)步骤，确保客户端以固定次数的数据库查询完成操作（支持递归处理）。

例如给定GraphQL查询：

```graphql
query {
  users(first: 100) {
    edges {
      node {
        photos {
          link
        }
        posts {
          content
          comments {
            content
          }
        }
      }
    }
  }
}
```

客户端将执行：1次查询获取用户，1次获取照片，2次获取帖子及其评论（共4次）。该逻辑同时适用于根查询/解析器和节点API。

### Schema配置

通过`entgql.Annotation`为特定边关系启用此功能：

```go
func (Todo) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("children", Todo.Type).
            Annotations(entgql.Bind()).
            From("parent").
            // Bind implies the edge name in graphql schema is
            // equivalent to the name used in ent schema.
            Annotations(entgql.Bind()).
            Unique(),
        edge.From("owner", User.Type).
            Ref("tasks").
            // Map edge names as defined in graphql schema.
            Annotations(entgql.MapsTo("taskOwner")),
    }
}
```

### 使用与配置

GraphQL扩展还会在`gql_edge.go`中生成节点边解析器：

```go
func (t *Todo) Children(ctx context.Context) ([]*Todo, error) {
	result, err := t.Edges.ChildrenOrErr()
	if IsNotLoaded(err) {
		result, err = t.QueryChildren().All(ctx)
	}
	return result, err
}
```

如需手动编写解析器，可在GraphQL schema中添加[`forceResolver`](https://gqlgen.com/master/config#inline-config-with-directives)选项：

```graphql
type Todo implements Node {
  id: ID!
  children: [Todo]! @goField(forceResolver: true)
}
```

随后可在类型解析器中实现该功能。

```go
func (r *todoResolver) Children(ctx context.Context, obj *ent.Todo) ([]*ent.Todo, error) {
	// Do something here.
	return obj.Edges.ChildrenOrErr()
}
```

## 枚举实现

枚举模板为ent生成的枚举实现了MarshalGQL/UnmarshalGQL方法。

## 事务性变更

`entgql.Transactioner` 处理器会将每个 GraphQL 变更操作置于事务中执行。注入到解析器的客户端是一个[事务型 `ent.Client`](transactions.md#transactional-client)。因此，使用 `ent.Client` 的代码无需修改。具体使用步骤如下：

1\. 在 GraphQL 服务器初始化时，按如下方式使用 `entgql.Transactioner` 处理器：

```go
srv := handler.NewDefaultServer(todo.NewSchema(client))
srv.Use(entgql.Transactioner{TxOpener: client})
```

2\. 随后，在 GraphQL 变更操作中，通过上下文获取客户端如下：

```go
func (mutationResolver) CreateTodo(ctx context.Context, todo TodoInput) (*ent.Todo, error) {
	client := ent.FromContext(ctx)
	return client.Todo.
		Create().
		SetStatus(todo.Status).
		SetNillablePriority(todo.Priority).
		SetText(todo.Text).
		SetNillableParentID(todo.Parent).
		Save(ctx)
}
```

## 示例

[ent/contrib](https://github.com/ent/contrib) 仓库目前包含以下典型示例：

1. 完整的 GraphQL 服务示例 - 使用数字 ID 字段的[待办应用](https://github.com/ent/contrib/tree/master/entgql/internal/todo)
2. 与示例 1 相同的[待办应用](https://github.com/ent/contrib/tree/master/entgql/internal/todouuid)，但采用 UUID 类型作为 ID 字段
3. 在示例 1 和 2 基础上，使用带前缀的 [ULID](https://github.com/ulid/spec)（即 `PULID`）作为 ID 字段的[待办应用](https://github.com/ent/contrib/tree/master/entgql/internal/todopulid)。该示例通过为 ID 添加实体类型前缀（而非采用[通用 ID](migrate.md#universal-ids) 中的 ID 空间分区方案）来实现 Relay Node API 支持。

---

请注意本文档正在持续完善中。所有代码实现位于 [ent/contrib/entgql](https://github.com/ent/contrib/tree/master/entgql)，完整的待办应用示例可参考 [ent/contrib/entgql/todo](https://github.com/ent/contrib/tree/master/entgql/internal/todo)。