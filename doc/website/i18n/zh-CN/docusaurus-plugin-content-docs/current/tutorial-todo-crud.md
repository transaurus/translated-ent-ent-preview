---
id: tutorial-todo-crud
title: Query and Mutation
sidebar_label: Query and Mutation
---

完成项目设置后，我们已准备好创建待办事项列表并进行查询。

## 创建待办事项

让我们在测试示例中创建一个待办事项。通过将以下代码添加到`example_test.go`中实现：

```go
func Example_Todo() {
	// ...
	task1, err := client.Todo.Create().Save(ctx)
	if err != nil {
		log.Fatalf("failed creating a todo: %v", err)
	}
	fmt.Println(task1)
	// Output:
	// Todo(id=1)
}
```

运行`go test`应能成功通过。

## 向Schema添加字段

如你所见，当前的待办事项仅包含`ID`字段过于简单。我们通过在`todo/ent/schema/todo.go`中添加多个字段来改进示例：

```go
func (Todo) Fields() []ent.Field {
	return []ent.Field{
		field.Text("text").
			NotEmpty(),
		field.Time("created_at").
			Default(time.Now).
			Immutable(),
		field.Enum("status").
			NamedValues(
				"InProgress", "IN_PROGRESS",
				"Completed", "COMPLETED",
			).
			Default("IN_PROGRESS"),
		field.Int("priority").
			Default(0),
	}
}
```

添加这些字段后，需要像之前一样运行代码生成：

```console
go generate ./ent
```

你可能注意到，除必须由用户提供的`text`字段外，所有字段在创建时都有默认值。我们修改`example_test.go`以适应这些变更：

```go
func Example_Todo() {
	// ...
	task1, err := client.Todo.Create().SetText("Add GraphQL Example").Save(ctx)
	if err != nil {
		log.Fatalf("failed creating a todo: %v", err)
	}
	fmt.Printf("%d: %q\n", task1.ID, task1.Text)
	task2, err := client.Todo.Create().SetText("Add Tracing Example").Save(ctx)
	if err != nil {
		log.Fatalf("failed creating a todo: %v", err)
	}
	fmt.Printf("%d: %q\n", task2.ID, task2.Text)
    // Output:
    // 1: "Add GraphQL Example"
    // 2: "Add Tracing Example"
}
```

太棒了！我们已在数据库中创建包含5列（`id`、`text`、`created_at`、`status`、`priority`）的schema，并通过向表插入2行数据创建了2个待办事项。

![tutorial-todo-create](https://entgo.io/images/assets/tutorial-todo-create-items.png)

## 向Schema添加边关系

假设我们需要设计待办事项间的依赖关系。因此我们为每个待办事项添加`parent`边以获取其依赖项，同时添加反向引用边`children`来获取所有依赖它的项。

再次修改`todo/ent/schema/todo.go`中的schema：

```go
func (Todo) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("parent", Todo.Type).
			Unique().
			From("children"),
	}
}
```

添加这些边关系后，需要像之前一样运行代码生成：

```console
go generate ./ent
```

## 建立待办事项关联

我们通过更新刚创建的两个待办事项继续边关系示例。设定item-2（*"添加追踪示例"*）依赖于item-1（*"添加GraphQL示例"*）。

![tutorial-todo-create](https://entgo.io/images/assets/tutorial-todo-create-edges.png)

```go
func Example_Todo() {
	// ...
	if err := task2.Update().SetParent(task1).Exec(ctx); err != nil {
		log.Fatalf("failed connecting todo2 to its parent: %v", err)
	}
    // Output:
    // 1: "Add GraphQL Example"
    // 2: "Add Tracing Example"
}
```

## 查询待办事项

建立item-2与item-1的关联后，现在可以开始查询待办事项列表。

#### 查询所有待办事项：

```go
func Example_Todo() {
	// ...

	// Query all todo items.
	items, err := client.Todo.Query().All(ctx)
	if err != nil {
		log.Fatalf("failed querying todos: %v", err)
	}
	for _, t := range items {
		fmt.Printf("%d: %q\n", t.ID, t.Text)
	}
	// Output:
	// 1: "Add GraphQL Example"
	// 2: "Add Tracing Example"
}
```

#### 查询所有依赖其他事项的待办事项：

```go
func Example_Todo() {
	// ...

	// Query all todo items that depend on other items.
	items, err := client.Todo.Query().Where(todo.HasParent()).All(ctx)
	if err != nil {
		log.Fatalf("failed querying todos: %v", err)
	}
	for _, t := range items {
		fmt.Printf("%d: %q\n", t.ID, t.Text)
	}
	// Output:
	// 2: "Add Tracing Example"
}
```

#### 查询所有不依赖其他事项且被其他事项依赖的待办事项：

```go
func Example_Todo() {
	// ...

	// Query all todo items that don't depend on other items and have items that depend them.
	items, err := client.Todo.Query().
		Where(
			todo.Not(
				todo.HasParent(),
			),
			todo.HasChildren(),
		).
		All(ctx)
	if err != nil {
		log.Fatalf("failed querying todos: %v", err)
	}
	for _, t := range items {
		fmt.Printf("%d: %q\n", t.ID, t.Text)
	}
	// Output:
	// 1: "Add GraphQL Example"
}
```

#### 通过子项查询父项：

```go
func Example_Todo() {
	// ...
	
	// Get a parent item through its children and expect the
	// query to return exactly one item.
	parent, err := client.Todo.Query(). // Query all todos.
		Where(todo.HasParent()).        // Filter only those with parents.
		QueryParent().                  // Continue traversals to the parents.
		Only(ctx)                       // Expect exactly one item.
	if err != nil {
		log.Fatalf("failed querying todos: %v", err)
	}
	fmt.Printf("%d: %q\n", parent.ID, parent.Text)
	// Output:
	// 1: "Add GraphQL Example"
}
```