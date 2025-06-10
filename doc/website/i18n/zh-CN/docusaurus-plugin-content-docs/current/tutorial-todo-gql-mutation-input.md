---
id: tutorial-todo-gql-mutation-input
title: Mutation Inputs
sidebar_label: Mutation Inputs
---

本节我们将通过扩展Ent代码生成器，使用Go模板为GraphQL突变生成可直接应用于Ent突变的[输入类型](https://graphql.org/graphql-js/mutations-and-input-types/)对象，继续完善[GraphQL示例](tutorial-todo-gql.mdx)。

#### 克隆代码（可选）

本教程代码托管于[github.com/a8m/ent-graphql-example](https://github.com/a8m/ent-graphql-example)，每个步骤都对应Git标签。若想跳过基础配置直接使用GraphQL服务器的初始版本，可按以下方式克隆仓库并运行程序：

```console
git clone git@github.com:a8m/ent-graphql-example.git
cd ent-graphql-example 
go run ./cmd/todo/
```

## 突变类型

Ent支持生成突变类型。这些类型可作为GraphQL突变的输入，由Ent进行处理和验证。以下配置声明我们的GraphQL `Todo`类型支持创建和更新操作：

```go title="ent/schema/todo.go"
func (Todo) Annotations() []schema.Annotation {
	return []schema.Annotation{
		entgql.QueryField(),
		//highlight-next-line
		entgql.Mutations(entgql.MutationCreate(), entgql.MutationUpdate()),
	}
}
```

运行代码生成命令：

```go
go generate .
```

您将注意到Ent生成了两种类型：`ent.CreateTodoInput`和`ent.UpdateTodoInput`。

## 突变实现

生成突变输入类型后，可将其与GraphQL突变进行绑定：

```graphql title="todo.graphql"
type Mutation {
  createTodo(input: CreateTodoInput!): Todo!
  updateTodo(id: ID!, input: UpdateTodoInput!): Todo!
}
```

运行代码生成将产生实际突变，最后只需将解析器与Ent绑定即可。

```go
go generate .
```

```go title="todo.resolvers.go"
// CreateTodo is the resolver for the createTodo field.
func (r *mutationResolver) CreateTodo(ctx context.Context, input ent.CreateTodoInput) (*ent.Todo, error) {
	return r.client.Todo.Create().SetInput(input).Save(ctx)
}

// UpdateTodo is the resolver for the updateTodo field.
func (r *mutationResolver) UpdateTodo(ctx context.Context, id int, input ent.UpdateTodoInput) (*ent.Todo, error) {
	return r.client.Todo.UpdateOneID(id).SetInput(input).Save(ctx)
}
```

## 测试`CreateTodo`解析器

首先通过执行两次`createTodo`突变来创建2个待办事项。

#### 突变语句

```graphql
mutation CreateTodo {
   createTodo(input: {text: "Create GraphQL Example", status: IN_PROGRESS, priority: 2}) {
     id
     text
     createdAt
     priority
     parent {
       id
     }
   }
 }
```

#### 输出结果

```json
{
  "data": {
    "createTodo": {
      "id": "1",
      "text": "Create GraphQL Example",
      "createdAt": "2021-04-19T10:49:52+03:00",
      "priority": 2,
      "parent": null
    }
  }
}
```

#### 突变语句

```graphql
mutation CreateTodo {
   createTodo(input: {text: "Create Tracing Example", status: IN_PROGRESS, priority: 2}) {
     id
     text
     createdAt
     priority
     parent {
       id
     }
   }
 }
```

#### 输出结果

```json
{
  "data": {
    "createTodo": {
      "id": "2",
      "text": "Create Tracing Example",
      "createdAt": "2021-04-19T10:50:01+03:00",
      "priority": 2,
      "parent": null
    }
  }
}
```

## 测试`UpdateTodo`解析器

最后测试`UpdateTodo`解析器，将第二个待办事项的`parent`字段更新为`1`。

```graphql
mutation UpdateTodo {
  updateTodo(id: 2, input: {parentID: 1}) {
    id
    text
    createdAt
    priority
    parent {
      id
      text
    }
  }
}
```

#### 输出结果

```json
{
  "data": {
    "updateTodo": {
      "id": "2",
      "text": "Create Tracing Example",
      "createdAt": "2021-04-19T10:50:01+03:00",
      "priority": 1,
      "parent": {
        "id": "1",
        "text": "Create GraphQL Example"
      }
    }
  }
}
```

## 通过突变创建边关系

若要在同一次突变中创建节点的边关系，可扩展GQL突变输入的边字段：

```graphql title="extended.graphql"
extend input CreateTodoInput {
  createChildren: [CreateTodoInput!]
}
```

再次运行代码生成：

```go
go generate .
```

GQLGen将生成`createChildren`字段的解析器，您可以在解析器中调用：

```go title="extended.resolvers.go"
// CreateChildren is the resolver for the createChildren field.
func (r *createTodoInputResolver) CreateChildren(ctx context.Context, obj *ent.CreateTodoInput, data []*ent.CreateTodoInput) error {
	panic(fmt.Errorf("not implemented: CreateChildren - createChildren"))
}
```

现在需要实现创建子节点的逻辑：

```go title="extended.resolvers.go"
// CreateChildren is the resolver for the createChildren field.
func (r *createTodoInputResolver) CreateChildren(ctx context.Context, obj *ent.CreateTodoInput, data []*ent.CreateTodoInput) error {
	// highlight-start
	// NOTE: We need to use the Ent client from the context.
	// To ensure we create all of the children in the same transaction.
	// See: Transactional Mutations for more information.
	c := ent.FromContext(ctx)
	// highlight-end
	builders := make([]*ent.TodoCreate, len(data))
	for i := range data {
		builders[i] = c.Todo.Create().SetInput(*data[i])
	}
	todos, err := c.Todo.CreateBulk(builders...).Save(ctx)
	if err != nil {
		return err
	}
	ids := make([]int, len(todos))
	for i := range todos {
		ids[i] = todos[i].ID
	}
	obj.ChildIDs = append(obj.ChildIDs, ids...)
	return nil
}
```

修改以下代码使用事务型客户端：

```go title="todo.resolvers.go"
// CreateTodo is the resolver for the createTodo field.
func (r *mutationResolver) CreateTodo(ctx context.Context, input ent.CreateTodoInput) (*ent.Todo, error) {
	// highlight-next-line
	return ent.FromContext(ctx).Todo.Create().SetInput(input).Save(ctx)
}

// UpdateTodo is the resolver for the updateTodo field.
func (r *mutationResolver) UpdateTodo(ctx context.Context, id int, input ent.UpdateTodoInput) (*ent.Todo, error) {
	// highlight-next-line
	return ent.FromContext(ctx).Todo.UpdateOneID(id).SetInput(input).Save(ctx)
}
```

测试带子节点的突变：

**突变语句**

```graphql
mutation {
  createTodo(input: {
    text: "parent", status:IN_PROGRESS,
    createChildren: [
      { text: "children1", status: IN_PROGRESS },
      { text: "children2", status: COMPLETED }
    ]
  }) {
    id
    text
    children {
      id
      text
      status
    }
  }
}
```

**输出结果**

```json
{
  "data": {
    "createTodo": {
      "id": "3",
      "text": "parent",
      "children": [
        {
          "id": "1",
          "text": "children1",
          "status": "IN_PROGRESS"
        },
        {
          "id": "2",
          "text": "children2",
          "status": "COMPLETED"
        }
      ]
    }
  }
}
```

若启用调试客户端，可见子节点创建于同一事务中：

```log
2022/12/14 00:27:41 driver.Tx(7e04b00b-7941-41c5-9aee-41c8c2d85312): started
2022/12/14 00:27:41 Tx(7e04b00b-7941-41c5-9aee-41c8c2d85312).Query: query=INSERT INTO `todos` (`created_at`, `priority`, `status`, `text`) VALUES (?, ?, ?, ?), (?, ?, ?, ?) RETURNING `id` args=[2022-12-14 00:27:41.046344 +0700 +07 m=+5.283557793 0 IN_PROGRESS children1 2022-12-14 00:27:41.046345 +0700 +07 m=+5.283558626 0 COMPLETED children2]
2022/12/14 00:27:41 Tx(7e04b00b-7941-41c5-9aee-41c8c2d85312).Query: query=INSERT INTO `todos` (`text`, `created_at`, `status`, `priority`) VALUES (?, ?, ?, ?) RETURNING `id` args=[parent 2022-12-14 00:27:41.047455 +0700 +07 m=+5.284669251 IN_PROGRESS 0]
2022/12/14 00:27:41 Tx(7e04b00b-7941-41c5-9aee-41c8c2d85312).Exec: query=UPDATE `todos` SET `todo_parent` = ? WHERE `id` IN (?, ?) AND `todo_parent` IS NULL args=[3 1 2]
2022/12/14 00:27:41 Tx(7e04b00b-7941-41c5-9aee-41c8c2d85312).Query: query=SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` WHERE `todo_parent` = ? args=[3]
2022/12/14 00:27:41 Tx(7e04b00b-7941-41c5-9aee-41c8c2d85312): committed
```