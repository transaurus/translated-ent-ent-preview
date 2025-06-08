---
title: Introducing ent
author: Ariel Mashraki
authorURL: "https://github.com/a8m"
authorImageURL: "https://avatars0.githubusercontent.com/u/7413593"
authorTwitter: arielmashraki
---

## Go语言在Facebook Connectivity特拉维夫团队的现状

20个月前，在积累了约5年Go语言开发经验并将其引入多家公司后，我加入了Facebook Connectivity（FBC）特拉维夫团队。  
当时我们团队正在启动新项目，需要选择编程语言。经过多语言对比后，我们最终选择了Go。

此后，Go语言在FBC其他项目中持续扩散，仅特拉维夫团队就已有约15名Go工程师，取得了巨大成功。**新服务现在均采用Go编写**。

## 开发新Go语言ORM的动机

在加入Facebook前的5年里，我的工作主要涉及基础设施工具和微服务开发，较少涉及数据模型领域。对于需要少量SQL数据库操作的服务，我们使用现有开源解决方案；但对于复杂数据模型的服务，则采用具备成熟ORM的其他语言实现，例如Python+SQLAlchemy组合。

在Facebook，我们习惯用图结构思维建模数据，这种模式在内部实践中表现优异。  
由于Go生态缺乏完善的图结构ORM，我们基于以下原则自主开发：

- **代码即Schema** - 类型定义、关系约束应通过Go代码（而非结构体标签）实现，并通过CLI工具验证。Facebook内部类似工具已有成功实践
- **静态类型化显式API** - 通过代码生成避免遍地`interface{}`，这对开发者效率（尤其新成员）至关重要
- **简化查询/聚合/图遍历** - 开发者不应处理原生SQL语句或术语
- **谓词需静态类型化** - 杜绝字符串满天飞
- 完整支持`context.Context` - 这对实现全链路追踪、日志系统及取消操作等特性不可或缺
- **存储无关性** - 通过代码生成模板保持存储层动态化，项目初期支持Gremlin（AWS Neptune），后扩展至MySQL

## 开源ent框架

**ent**是基于上述原则构建的Go实体框架（ORM）。通过**ent**，开发者可以轻松用Go代码定义任意数据模型或图结构。Schema配置由**entc**（ent代码生成器）验证，并生成符合Go语言习惯的静态类型API，保持开发效率与愉悦度。当前支持MySQL、MariaDB、PostgreSQL、SQLite及基于Gremlin的图数据库。

今天我们正式开源**ent**，欢迎开始使用 → [entgo.io/docs/getting-started](/docs/getting-started)