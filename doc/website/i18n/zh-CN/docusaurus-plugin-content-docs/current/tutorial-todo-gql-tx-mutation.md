---
id: tutorial-todo-gql-tx-mutation
title: Transactional Mutations
sidebar_label: Transactional Mutations
---

本节我们将延续[GraphQL示例](tutorial-todo-gql.mdx)，讲解如何将GraphQL变更加入事务处理机制。这意味着系统会自动用数据库事务包裹GraphQL变更操作，最终执行提交或在发生GraphQL错误时回滚事务。

#### 克隆代码（可选）

本教程代码托管于[github.com/a8m/ent-graphql-example](https://github.com/a8m/ent-graphql-example)，每个步骤都有对应的Git标签。若想跳过基础配置直接使用GraphQL服务器的初始版本，可克隆仓库并检出`v0.1.0`版本：

```console
git clone git@github.com:a8m/ent-graphql-example.git
cd ent-graphql-example 
go run ./cmd/todo/
```

## 使用方式

GraphQL扩展提供了名为`entgql.Transactioner`的处理器，该处理器会为每个GraphQL变更操作创建事务。注入到解析器的客户端是[事务型ent.Client](transactions.md#transactional-client)，因此使用`ent.Client`的GraphQL解析器无需修改。要为待办事项应用添加此功能，请按以下步骤操作：

1\. 编辑`cmd/todo/main.go`文件，在GraphQL服务器初始化时加入`entgql.Transactioner`处理器：

```diff title="cmd/todo/main.go"
srv := handler.NewDefaultServer(todo.NewSchema(client))
+srv.Use(entgql.Transactioner{TxOpener: client})
```

2\. 在GraphQL变更操作中，通过上下文获取客户端实例：

```diff title="todo.resolvers.go"
}
+func (mutationResolver) CreateTodo(ctx context.Context, input ent.CreateTodoInput) (*ent.Todo, error) {
+	client := ent.FromContext(ctx)
+	return client.Todo.Create().SetInput(input).Save(ctx)
-func (r *mutationResolver) CreateTodo(ctx context.Context, input ent.CreateTodoInput) (*ent.Todo, error) {
-	return r.client.Todo.Create().SetInput(input).Save(ctx)
}
```

## 隔离级别

如需调整事务隔离级别，可通过实现自定义`TxOpener`来实现。例如：

```go title="cmd/todo/main.go"
srv.Use(entgql.Transactioner{
	TxOpener: entgql.TxOpenerFunc(func(ctx context.Context) (context.Context, driver.Tx, error) {
		tx, err := client.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelRepeatableRead})
		if err != nil {
			return nil, nil, err
		}
		ctx = ent.NewTxContext(ctx, tx)
		ctx = ent.NewContext(ctx, tx.Client())
		return ctx, tx, nil
	}),
})
```

## 跳过事务

默认情况下`entgql.Transactioner`会为所有变更操作启用事务。但某些变更操作可能不需要数据库访问或需要特殊处理。此时可通过设置自定义`SkipTxFunc`函数或使用内置函数来跳过事务处理。

```go title="cmd/todo/main.go" {4,10,16-18}
srv.Use(entgql.Transactioner{
	TxOpener: client,
	// Skip the given operation names from running under a transaction.
	SkipTxFunc: entgql.SkipOperations("operation1", "operation2"),
})

srv.Use(entgql.Transactioner{
	TxOpener: client,
	// Skip if the operation has a mutation field with the given names.
	SkipTxFunc: entgql.SkipIfHasFields("field1", "field2"),
})

srv.Use(entgql.Transactioner{
	TxOpener: client,
	// Custom skip function.
	SkipTxFunc: func(*ast.OperationDefinition) bool {
	    // ...
    },
})
```

---

太棒了！仅需几行代码，我们的应用就实现了自动化事务变更。接下来请继续阅读下一章节，我们将讲解如何扩展Ent代码生成器来为GraphQL变更操作生成[GraphQL输入类型](https://graphql.org/graphql-js/mutations-and-input-types/)。