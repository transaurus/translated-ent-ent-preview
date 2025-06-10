---
id: tutorial-todo-gql-node
title: Relay Node Interface
sidebar_label: Relay Node Interface
---

本节我们将通过实现 [Relay Node Interface](https://relay.dev/graphql/objectidentification.htm) 来继续完善 [GraphQL 示例](tutorial-todo-gql.mdx)。若您不熟悉 Node 接口，请阅读以下摘自 [relay.dev](https://relay.dev/graphql/objectidentification.htm#sel-DABDDBAADLA0Cl0c) 的说明：

> 为便于 GraphQL 客户端优雅地处理缓存和数据重取，GraphQL 服务端需以标准化方式暴露对象标识符。在查询中，模式应提供通过ID获取对象的标准机制；在响应中，模式需标准化提供这些ID。
>
> 我们将具有标识符的对象称为"节点"。以下查询展示了这两种机制：
>
>  ```graphql
>   {
>       node(id: "4") {
>           id
>          ... on User {
>               name
>           }
>       }
>   }
> ```

#### 克隆代码（可选）

本教程代码存放于 [github.com/a8m/ent-graphql-example](https://github.com/a8m/ent-graphql-example)，每个步骤都对应 Git 标签。若想跳过基础配置直接使用 GraphQL 服务初始版本，可按以下方式克隆仓库：

```console
git clone git@github.com:a8m/ent-graphql-example.git
cd ent-graphql-example 
go run ./cmd/todo/
```

## 实现方案

Ent 通过其 GraphQL 集成原生支持 Node 接口。只需简单几步即可为应用添加支持。首先编辑 `gqlgen.yaml` 文件，声明 Ent 提供 `Node` 接口：

```diff title="gqlgen.yml" {7-9}
# This section declares type mapping between the GraphQL and Go type systems.
models:
  # Defines the ID field as Go 'int'.
  ID:
    model:
      - github.com/99designs/gqlgen/graphql.IntID
  Node:
    model:
      - todo/ent.Noder
```

应用变更需重新运行代码生成：

```console
go generate .
```

与之前类似，我们需要在 `ent.resolvers.go` 中实现 GraphQL 解析器。只需单行改动，将生成的 `gqlgen` 代码替换为：

```diff title="ent.resolvers.go"
func (r *queryResolver) Node(ctx context.Context, id int) (ent.Noder, error) {
-	panic(fmt.Errorf("not implemented: Node - node"))
+	return r.client.Noder(ctx, id)
}

func (r *queryResolver) Nodes(ctx context.Context, ids []int) ([]ent.Noder, error) {
-	panic(fmt.Errorf("not implemented: Nodes - nodes"))
+	return r.client.Noders(ctx, ids)
}
```

## 查询节点

现在可以测试新的 GraphQL 解析器了。首先通过多次运行以下查询创建待办事项（修改变量可选）：

```graphql
mutation CreateTodo($input: CreateTodoInput!) {
    createTodo(input: $input) {
        id
        text
        createdAt
        priority
        parent {
            id
        }
    }
}

# Query Variables: { "input": { "text":"Create GraphQL Example", "status": "IN_PROGRESS", "priority": 1 } }
# Output: { "data": { "createTodo": { "id": "2", "text": "Create GraphQL Example", "createdAt": "2021-03-10T15:02:18+02:00", "priority": 1, "parent": null } } }
```

对某个待办事项执行 **Node** API 将返回：

````graphql
query {
  node(id: 1) {
    id
    ... on Todo {
      text
    }
  }
}

# Output: { "data": { "node": { "id": "1", "text": "Create GraphQL Example" } } }
````

对多个待办事项执行 **Nodes** API 将返回：

```graphql
query {
  nodes(ids: [1, 2]) {
    id
    ... on Todo {
      text
    }
  }
}

# Output: { "data": { "nodes": [ { "id": "1", "text": "Create GraphQL Example" }, { "id": "2", "text": "Create Tracing Example" } ] } }
```

---

完成！如您所见，仅需少量代码改动，我们的应用就实现了 Relay Node 接口。下一节我们将展示如何用 Ent 实现 Relay 游标连接规范，这对支持查询结果切片与分页非常有用。