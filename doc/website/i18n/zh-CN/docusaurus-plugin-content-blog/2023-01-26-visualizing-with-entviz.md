---
title: Quickly visualize your Ent schemas with entviz
author: Rotem Tamir
authorURL: "https://github.com/rotemtam"
authorImageURL: "https://s.gravatar.com/avatar/36b3739951a27d2e37251867b7d44b1a?s=80"
authorTwitter: _rtam
image: "https://entgo.io/images/assets/entviz-v2.png"
---

### 快速概览

要获取Ent架构的可视化公开链接，请运行：

```
go run -mod=mod ariga.io/entviz ./path/to/ent/schema 
```

![](https://entgo.io/images/assets/erd/edges-quick-summary.png)

### 可视化Ent架构

Ent允许开发者使用[图语义](https://en.wikipedia.org/wiki/Graph_theory)构建复杂的应用数据模型：
不同于定义表、列、关联表和主外键，Ent模型仅通过[节点](https://entgo.io/docs/schema-fields)
和[边](https://entgo.io/docs/schema-edges)来定义：

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
)

// User schema.
type User struct {
	ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		// ...
	}
}

// Edges of the user.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("pets", Pet.Type),
	}
}
```

这种建模方式具有诸多优势，例如能够通过直观的API轻松[遍历](https://entgo.io/docs/traversals)应用数据图、
自动生成[GraphQL](https://entgo.io/docs/tutorial-todo-gql)服务等。

虽然Ent可以使用图数据库作为存储层，但大多数用户仍选择MySQL、PostgreSQL或MariaDB等关系型数据库。在这些场景中，开发者常会思考：*Ent最终会为我的应用架构生成怎样的数据库表结构？*

无论您是刚接触Ent需要学习基础建模，还是专家级用户需要优化数据库性能，能够直观查看Ent架构对应的底层数据库结构都极具价值。

#### 全新`entviz`工具

一年半前我们曾[分享过名为entviz的扩展](https://entgo.io/blog/2021/08/26/visualizing-your-data-graph-using-entviz)，
该工具可生成包含实体关系图的本地HTML文档来描述Ent架构。

如今，我们很高兴展示由[crossworth](https://github.com/crossworth)开发的[全新工具](https://github.com/ariga/entviz)，
它通过简单的Go命令即可实现：

```
go run -mod=mod ariga.io/entviz ./path/to/ent/schema 
```

该工具会分析您的Ent架构，在[Atlas Playground](https://gh.atlasgo.cloud)生成可视化图表，
并创建可分享的公开[链接](https://gh.atlasgo.cloud/explore/saved/60129542154)：

```
Here is a public link to your schema visualization:
	    https://gh.atlasgo.cloud/explore/saved/60129542154
```

通过该链接，您既可以查看ERD形式的关系图，也可以查看SQL或[Atlas HCL](https://atlasgo.io/atlas-schema/sql-resources)格式的文本化架构。

### 结语

本文探讨了需要快速可视化Ent架构的常见场景，并演示了如何使用[entviz](https://github.com/ariga/entviz)实现这一需求。如果您喜欢这个工具，欢迎立即试用并反馈意见！

:::note[获取更多Ent资讯：]

- 订阅我们的[新闻简报](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)
  :::