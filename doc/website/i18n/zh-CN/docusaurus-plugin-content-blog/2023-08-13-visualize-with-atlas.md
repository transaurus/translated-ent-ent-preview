---
title: "Quickly Generate ERDs from your Ent Schemas (Updated)" 
author: Rotem Tamir
authorURL: "https://github.com/rotemtam"
authorImageURL: "https://s.gravatar.com/avatar/36b3739951a27d2e37251867b7d44b1a?s=80"
authorTwitter: _rtam
image: "https://atlasgo.io/uploads/ent/inspect/entviz.png"
---

### 内容提要

只需一条命令即可生成Ent模式的可视化图表：

```
atlas schema inspect \
  -u ent://ent/schema \
  --dev-url "sqlite://demo?mode=memory&_fk=1" \
  --visualize
```

![](https://entgo.io/images/assets/erd/edges-quick-summary.png)

大家好！

数月前我们发布了[entviz](/blog/2023/01/26/visualizing-with-entviz)，这个工具能帮助您可视化Ent模式。鉴于其成功与受欢迎程度，我们决定将其直接集成至Ent使用的迁移引擎[Atlas](https://atlasgo.io)中。

自Atlas[v0.13.0](https://atlasgo.io/blog/2023/08/06/atlas-v-0-13)版本发布后，您现在可以直接通过Atlas可视化Ent模式，无需额外安装工具。

### 私有与公开可视化

此前您只能将模式可视化图表分享至[Atlas公共沙盒](https://gh.atlasgo.cloud/explore)。虽然便于与他人共享模式，但对于许多维护敏感模式的团队而言，这种方式无法接受。

新版本允许您直接将模式发布至[Atlas云平台](https://atlasgo.cloud)的私有工作区，确保仅您和团队成员可访问模式可视化图表。

### 使用Atlas可视化Ent模式

首先安装最新版Atlas：

```
curl -sSfL https://atlasgo.io/install.sh | sh
```

其他安装方式请参阅[Atlas安装文档](https://atlasgo.io/getting-started#installation)。

运行以下命令生成Ent模式可视化：

```
atlas schema inspect \
  -u ent://ent/schema \
  --dev-url "sqlite://demo?mode=memory&_fk=1" \
  --visualize
```

命令解析：

* `atlas schema inspect` - 该命令可检查多种来源的模式并以不同格式输出。此处我们用它检查Ent模式。
* `-u ent://ent/schema` - 指定待检查的Ent模式URL。本例使用`ent://`模式加载器指向本地`./ent/schema`目录。
* `--dev-url "sqlite://demo?mode=memory&_fk=1"` - Atlas依赖名为[开发数据库](https://atlasgo.io/concepts/dev-database)的空数据库进行模式标准化计算。本例使用内存SQLite数据库；若使用其他驱动，可替换为`docker://mysql/8/dev`（MySQL）或`docker://postgres/15/?search_path=public`（PostgreSQL）。

命令执行后将输出：

```text
Use the arrow keys to navigate: ↓ ↑ → ←
? Where would you like to share your schema visualization?:
  ▸ Publicly (gh.atlasgo.cloud)
    Your personal workspace (requires 'atlas login')
```

选择第一项可公开分享模式；选择第二项后运行`atlas login`登录（免费）Atlas账号即可私有分享。

### 总结

本文演示了如何通过Atlas轻松可视化Ent模式。期待该功能为您带来便利，欢迎反馈意见！

:::note[获取更多Ent资讯：]

- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::