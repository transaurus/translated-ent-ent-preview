---
title: Announcing preview support for TiDB
author: Amit Shani
authorURL: "https://github.com/hedwigz"
authorImageURL: "https://avatars.githubusercontent.com/u/8277210?v=4"
authorTwitter: itsamitush
---

我们[此前已宣布](2022-01-20-announcing-new-migration-engine.md)Ent的新迁移引擎Atlas。通过Atlas，为Ent添加对新数据库的支持变得前所未有的简单。今天，我很高兴地宣布，通过启用Atlas的最新版Ent，现已提供对[TiDB](https://en.pingcap.com/tidb/)的预览支持。

Ent可用于访问多种类型数据库中的数据，包括图数据库和关系型数据库。最常见的是，用户一直在使用标准的开源关系型数据库，如MySQL、MariaDB和PostgreSQL。随着基于Ent构建的应用程序团队日益成功，需要处理更大规模的流量时，这些单节点数据库往往成为扩展的瓶颈。因此，Ent社区的许多成员请求支持[NewSQL](https://en.wikipedia.org/wiki/NewSQL)数据库，如TiDB。

### TiDB

[TiDB](https://en.pingcap.com/tidb/)是一个[开源](https://github.com/pingcap/tidb)的NewSQL数据库。它提供了许多传统数据库不具备的特性，例如：

1. **水平扩展** - 多年来，软件架构师需要在关系型数据库提供的熟悉度和保证与_NoSQL_数据库（如MongoDB或Cassandra）的扩展能力之间做出选择。TiDB在保持良好MySQL特性兼容性的同时支持水平扩展。
2. **HTAP（混合事务/分析处理）** - 此外，数据库传统上分为分析型（OLAP）和事务型（OLTP）数据库。TiDB打破了这种二分法，支持在同一数据库上同时进行分析和事务工作负载。
3. **预置监控** w/ Prometheus+Grafana - TiDB从底层构建时就基于云原生范式，原生支持标准的CNCF可观测性堆栈。

要了解更多信息，请查看官方的[TiDB简介](https://docs.pingcap.com/tidb/stable)。

### TiDB的Hello World示例

要快速体验Ent+TiDB的"Hello World"应用程序，请按照以下步骤操作：

1. 使用Docker启动一个本地TiDB服务器：

```shell
 docker run -p 4000:4000 pingcap/tidb
 ```

现在你应该有一个运行中的TiDB实例在端口4000上监听。

2. 克隆示例[`hello world`仓库](https://github.com/hedwigz/tidb-hello-world)：

```shell
 git clone https://github.com/hedwigz/tidb-hello-world.git
 ```

在这个示例仓库中，我们定义了一个简单的`User`模式：

```go title="ent/schema/user.go"
 func (User) Fields() []ent.Field {
 	return []ent.Field{
 		field.Time("created_at").
 			Default(time.Now),
 		field.String("name"),
 		field.Int("age"),
 	}
 }
 ```

然后，我们将Ent连接到TiDB：

```go title="main.go"
 client, err := ent.Open("mysql", "root@tcp(localhost:4000)/test?parseTime=true")
 if err != nil {
 	log.Fatalf("failed opening connection to tidb: %v", err)
 }
 defer client.Close()
 // Run the auto migration tool, with Atlas.
 if err := client.Schema.Create(context.Background(), schema.WithAtlas(true)); err != nil {
 	log.Fatalf("failed printing schema changes: %v", err)
 }
	```

请注意，在第`1`行中，我们使用`mysql`方言连接到TiDB服务器。这是可能的，因为TiDB是[MySQL兼容的](https://docs.pingcap.com/tidb/stable/mysql-compatibility)，不需要任何特殊驱动程序。  
尽管如此，TiDB和MySQL之间存在一些差异，特别是在模式迁移方面，如信息模式检查和迁移规划。为此，`Atlas`会自动检测是否连接到`TiDB`并相应地处理迁移。  
此外，请注意在第`7`行中，我们使用了`schema.WithAtlas(true)`，这标志Ent使用`Atlas`作为其迁移引擎。

最后，我们创建一个用户并将记录保存到TiDB，以便稍后查询和打印。

```go title="main.go"
 client.User.Create().
		SetAge(30).
		SetName("hedwigz").
		SaveX(context.Background())
 user := client.User.Query().FirstX(context.Background())
 fmt.Printf("the user: %s is %d years old\n", user.Name, user.Age)
 ```

3. 运行示例程序：

```go
 $ go run main.go
 the user: hedwigz is 30 years old
 ```

哇！在这个快速演练中，我们成功实现了：

* 启动一个本地TiDB实例。
* 将Ent连接到TiDB。
* 使用Atlas迁移我们的Ent模式。
* 使用Ent插入和查询TiDB中的数据。

### 预览支持

当前Atlas与TiDB的集成已通过TiDB `v5.4.0`版本（撰写本文时为`latest`）的充分测试，未来我们将持续扩展支持范围。若您使用其他TiDB版本或需要协助，请随时[提交issue](https://github.com/ariga/atlas/issues)或加入我们的[Discord频道](https://discord.gg/zZ6sWVg6NT)。

:::note[获取更多Ent资讯：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter官方账号](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::