---
id: tutorial-todo-gql-field-collection
title: GraphQL Field Collection
sidebar_label: Field Collection
---

本节我们将通过解释Ent如何为GraphQL模式实现[字段收集](https://spec.graphql.org/June2018/#sec-Field-Collection)，继续完善[GraphQL示例](tutorial-todo-gql.mdx)，并解决解析器中的"N+1问题"。

#### 克隆代码（可选）

本教程代码托管在[github.com/a8m/ent-graphql-example](https://github.com/a8m/ent-graphql-example)，每个步骤都有对应的Git标签。若想跳过基础配置直接使用GraphQL服务器的初始版本，可通过以下命令克隆仓库：

```console
git clone git@github.com:a8m/ent-graphql-example.git
cd ent-graphql-example 
go run ./cmd/todo/
```

## 问题背景

GraphQL中的*"N+1问题"*指服务器为获取节点关联（即边关系）执行了不必要的数据库查询。潜在查询数量（N+1）是根查询返回的节点数量、其关联节点数量等递归因素的乘积，实际可能远大于N+1。

以下面查询为例说明：

```graphql
query {
    users(first: 50) {
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

该查询试图获取前50个用户及其照片、帖子（含评论）。

**在原始方案中**（问题场景），服务器会先执行1次查询获取50个用户，然后为每个用户执行：获取照片（50次）、获取帖子（50次）。假设每个用户有10篇帖子，则每篇帖子还需执行获取评论的查询（500次）。最终将产生`1+50+50+500=601`次查询。

![gql-请求树状图](https://entgo.io/images/assets/request-tree.png)

## Ent解决方案

Ent的字段收集扩展通过[预加载](eager-load.mdx)机制，为关联关系（即边）实现了自动化的[GraphQL字段收集](https://spec.graphql.org/June2018/#sec-Field-Collection)。当查询请求节点及其边关系时，`entgql`会自动在根查询中添加[`With<E>`](eager-load.mdx)步骤，最终客户端只需执行固定次数的数据库查询——该机制支持递归处理。

针对前文GraphQL查询，客户端仅需执行：1次获取用户、1次获取照片、2次获取帖子及评论（**总计4次！**）。此逻辑同时适用于根查询/解析器和节点API。

## 示例演示

为演示效果，我们**临时禁用自动字段收集**，在`Todos`解析器中启用`ent.Client`的调试模式并重启GraphQL服务：

```diff title="ent.resolvers.go"
func (r *queryResolver) Todos(ctx context.Context, after *ent.Cursor, first *int, before *ent.Cursor, last *int, orderBy *ent.TodoOrder) (*ent.TodoConnection, error) {
-	return r.client.Todo.Query().
+	return r.client.Debug().Todo.Query().
		Paginate(ctx, after, first, before, last,
			ent.WithTodoOrder(orderBy),
		)
}
```

执行[分页教程](tutorial-todo-gql-paginate.md)中的GraphQL查询，并添加`parent`边关系：

```graphql
query {
    todos(last: 10, orderBy: {direction: DESC, field: TEXT}) {
        edges {
            node {
                id
                text
                parent {
                    id
                }
            }
            cursor
        }
    }
}
```

观察输出可见服务器执行了11次查询：1次获取最近10条待办事项，另外10次分别获取每条事项的父项：

```sql
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` ORDER BY `id` ASC LIMIT 11
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
```

让我们看Ent如何自动解决问题：当定义Ent边时，`entgql`会自动绑定其在GraphQL中的使用，并在`gql_edge.go`文件中生成节点对应的边解析器：

```go title="ent/gql_edge.go"
func (t *Todo) Children(ctx context.Context) ([]*Todo, error) {
	if fc := graphql.GetFieldContext(ctx); fc != nil && fc.Field.Alias != "" {
		result, err = t.NamedChildren(graphql.GetFieldContext(ctx).Field.Alias)
	} else {
		result, err = t.Edges.ChildrenOrErr()
	}
	if IsNotLoaded(err) {
		result, err = t.QueryChildren().All(ctx)
	}
	return result, err
}
```

如果我们再次检查进程输出（此时**未禁用字段收集**），会发现服务器此次仅执行了两条数据库查询：第一条获取最近的10个待办事项，第二条批量获取这些待办事项各自的父项。

```sql
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority`, `todos`.`todo_parent` FROM `todos` ORDER BY `id` DESC LIMIT 11
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` WHERE `todos`.`id` IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
```

若运行示例时遇到问题，请跳转至[初始章节](#clone-the-code-optional)克隆代码并运行示例。

## 字段映射

[`entgql.MapsTo`](https://pkg.go.dev/entgo.io/contrib/entgql#MapsTo)支持在Ent模式与GraphQL模式间建立自定义字段/边映射。当需要以不同名称暴露字段或边时（例如下列场景）特别有用：

```go
// One to one mapping.
field.Int("priority").
	Annotations(
		entgql.OrderField("PRIORITY_ORDER"),
		entgql.MapsTo("priorityOrder"),
	)

// Multiple GraphQL fields can map to the same Ent field.
field.Int("category_id").
	Annotations(
		entgql.MapsTo("categoryID", "category_id", "categoryX"),
	)
```

---

成功！通过为Ent模式定义启用自动字段收集，我们显著提升了GraphQL查询效率。下一节将学习如何实现事务型GraphQL变更操作。