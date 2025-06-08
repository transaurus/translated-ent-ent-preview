---
title: Generating Ent Schemas from Existing SQL Databases 
author: Zeev Manilovich
authorURL: "https://github.com/zeevmoney"
authorImageURL: "https://avatars.githubusercontent.com/u/7361100?v=4"
---

数月前，Ent项目发布了[Schema Import Initiative](https://entgo.io/blog/2021/05/04/announcing-schema-imports)，其目标是支持从外部资源生成Ent模式的各种用例。今天，我很高兴分享一个我一直在开发的项目：**entimport**——一个"importent"（双关语）命令行工具，旨在从现有SQL数据库创建Ent模式。这是社区长期以来期待的功能，希望它能对许多人有所帮助。它可以帮助简化从其他语言或ORM迁移到Ent的现有设置，也可以用于需要从不同平台访问相同数据的场景（例如实现自动同步）。  
首个版本支持MySQL和PostgreSQL数据库（存在部分限制，下文详述），对其他关系型数据库（如SQLite）的支持正在开发中。

## 快速入门

为了让您了解`entimport`的工作原理，我将通过一个MySQL数据库的端到端示例进行演示。整体流程如下：

1. 创建数据库和模式 - 我们将先创建数据库并定义若干表，以展示`entimport`如何为现有数据库生成Ent模式
2. 初始化Ent项目 - 使用Ent CLI创建所需目录结构和Ent模式生成脚本
3. 安装`entimport`
4. 针对演示数据库运行`entimport` - 将我们创建的数据库模式导入Ent项目
5. 讲解如何配合生成的模式使用Ent

让我们开始吧。

### 创建数据库

首先创建数据库。我推荐使用[Docker](https://docs.docker.com/get-docker/)容器，通过`docker-compose`自动传递MySQL容器所需的所有参数。

在名为`entimport-example`的新目录中启动项目。创建`docker-compose.yaml`文件并粘贴以下内容：

```yaml
version: "3.7"

services:

  mysql8:
    platform: linux/amd64
    image: mysql
    environment:
      MYSQL_DATABASE: entimport
      MYSQL_ROOT_PASSWORD: pass
    healthcheck:
      test: mysqladmin ping -ppass
    ports:
      - "3306:3306"
```

该文件包含MySQL Docker容器的服务配置。使用以下命令运行：

```shell
docker-compose up -d
```

接下来创建简单模式。本示例将使用两个实体间的关系：

- 用户(User)
- 汽车(Car)

使用MySQL shell连接数据库，可执行以下命令：

> 请确保在项目根目录下运行

```shell
docker-compose exec mysql8 mysql --database=entimport -ppass
```

```sql
create table users
(
    id        bigint auto_increment primary key,
    age       bigint       not null,
    name      varchar(255) not null,
    last_name varchar(255) null comment 'surname'
);

create table cars
(
    id          bigint auto_increment primary key,
    model       varchar(255) not null,
    color       varchar(255) not null,
    engine_size mediumint    not null,
    user_id     bigint       null,
    constraint cars_owners foreign key (user_id) references users (id) on delete set null
);
```

验证是否已创建上述表，在MySQL shell中执行：

```sql
show tables;
+---------------------+
| Tables_in_entimport |
+---------------------+
| cars                |
| users               |
+---------------------+
```

应看到两个表：`users`和`cars`

### 初始化Ent项目

完成数据库和基础模式创建后，我们需要创建一个包含Ent的[Go](https://golang.org/doc/install)项目。本阶段将说明具体步骤，由于最终要使用导入的模式，因此需要创建Ent目录结构。

在`entimport-example`目录中初始化新的Go项目：

```shell
go mod init entimport-example
```

运行Ent初始化命令：

```shell
go run -mod=mod entgo.io/ent/cmd/ent new 
```

项目结构应如下所示：

```
├── docker-compose.yaml
├── ent
│   ├── generate.go
│   └── schema
└── go.mod
```

### 安装entimport

现在，有趣的环节开始了！我们终于可以安装`entimport`并见证它的实际运作了。  
首先运行`entimport`：

```shell
go run -mod=mod ariga.io/entimport/cmd/entimport -h
```

`entimport`将被下载，命令行会输出：

```
Usage of entimport:
  -dialect string
        database dialect (default "mysql")
  -dsn string
        data source name (connection information)
  -schema-path string
        output path for ent schema (default "./ent/schema")
  -tables value
        comma-separated list of tables to inspect (all if empty)
```

### 运行entimport

现在我们可以将MySQL模式导入到Ent了！

通过以下命令实现：

> 该命令会导入模式中的所有表，也可通过`-tables`标志限定特定表。

```shell
go run ariga.io/entimport/cmd/entimport -dialect mysql -dsn "root:pass@tcp(localhost:3306)/entimport"
```

与许多Unix工具类似，`entimport`在成功运行时不会输出任何信息。要验证运行是否正常，我们需要检查文件系统，特别是`ent/schema`目录。

```console {5-6}
├── docker-compose.yaml
├── ent
│   ├── generate.go
│   └── schema
│       ├── car.go
│       └── user.go
├── go.mod
└── go.sum
```

让我们看看结果——记得我们有两个模式：`users`模式和具有一对多关系的`cars`模式。现在来看看`entimport`的表现。

```go title="entimport-example/ent/schema/user.go"
type User struct {
	ent.Schema
}

func (User) Fields() []ent.Field {
	return []ent.Field{field.Int("id"), field.Int("age"), field.String("name"), field.String("last_name").Optional().Comment("surname")}
}
func (User) Edges() []ent.Edge {
	return []ent.Edge{edge.To("cars", Car.Type)}
}
func (User) Annotations() []schema.Annotation {
	return nil
}
```

```go title="entimport-example/ent/schema/car.go"
type Car struct {
	ent.Schema
}

func (Car) Fields() []ent.Field {
	return []ent.Field{field.Int("id"), field.String("model"), field.String("color"), field.Int32("engine_size"), field.Int("user_id").Optional()}
}
func (Car) Edges() []ent.Edge {
	return []ent.Edge{edge.From("user", User.Type).Ref("cars").Unique().Field("user_id")}
}
func (Car) Annotations() []schema.Annotation {
	return nil
}
```

> **`entimport`成功创建了实体及其关系！**

目前看起来不错，现在让我们实际试用一下。首先必须生成Ent模式。因为Ent是**模式优先**的ORM框架，它会为不同数据库[生成](https://entgo.io/docs/code-gen)交互用的Go代码。

运行Ent代码生成：

```shell
go generate ./ent
```

查看`ent`目录：

```
...
├── ent
│   ├── car
│   │   ├── car.go
│   │   └── where.go
...
│   ├── schema
│   │   ├── car.go
│   │   └── user.go
...
│   ├── user
│   │   ├── user.go
│   │   └── where.go
...
```

### Ent示例

快速运行示例验证模式是否生效：

在项目根目录创建`example.go`文件，内容如下：

> 该部分示例可在此[查看](https://github.com/zeevmoney/entimport-example/blob/master/part1/example.go)

```go title="entimport-example/example.go"
package main

import (
	"context"
	"fmt"
	"log"

	"entimport-example/ent"

	"entgo.io/ent/dialect"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	client, err := ent.Open(dialect.MySQL, "root:pass@tcp(localhost:3306)/entimport?parseTime=True")
	if err != nil {
		log.Fatalf("failed opening connection to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	example(ctx, client)
}
```

尝试添加用户，在文件末尾写入以下代码：

```go title="entimport-example/example.go"
func example(ctx context.Context, client *ent.Client) {
	// Create a User.
	zeev := client.User.
		Create().
		SetAge(33).
		SetName("Zeev").
		SetLastName("Manilovich").
		SaveX(ctx)
	fmt.Println("User created:", zeev)
}
```

然后运行：

```shell
go run example.go
```

预期输出：

`# 用户已创建: User(id=1, age=33, name=Zeev, last_name=Manilovich)`

通过数据库验证用户是否成功添加

```sql
SELECT *
FROM users
WHERE name = 'Zeev';

+--+---+----+----------+
|id|age|name|last_name |
+--+---+----+----------+
|1 |33 |Zeev|Manilovich|
+--+---+----+----------+
```

很好！现在让我们进一步使用Ent添加关系，在`example()`函数末尾追加代码：

> 确保在import()声明中添加`"entimport-example/ent/user"`

```go title="entimport-example/example.go"
// Create Car.
vw := client.Car.
    Create().
    SetModel("volkswagen").
    SetColor("blue").
    SetEngineSize(1400).
    SaveX(ctx)
fmt.Println("First car created:", vw)

// Update the user - add the car relation.
client.User.Update().Where(user.ID(zeev.ID)).AddCars(vw).SaveX(ctx)

// Query all cars that belong to the user.
cars := zeev.QueryCars().AllX(ctx)
fmt.Println("User cars:", cars)

// Create a second Car.
delorean := client.Car.
    Create().
    SetModel("delorean").
    SetColor("silver").
    SetEngineSize(9999).
    SaveX(ctx)
fmt.Println("Second car created:", delorean)

// Update the user - add another car relation.
client.User.Update().Where(user.ID(zeev.ID)).AddCars(delorean).SaveX(ctx)

// Traverse the sub-graph.
cars = delorean.
    QueryUser().
    QueryCars().
    AllX(ctx)
fmt.Println("User cars:", cars)
```

> 该部分示例可在此[查看](https://github.com/zeevmoney/entimport-example/blob/master/part2/example.go)

执行：`go run example.go`  
运行上述代码后，数据库中将存在一个用户与两辆汽车的O2M关系记录。

```sql
SELECT *
FROM users;

+--+---+----+----------+
|id|age|name|last_name |
+--+---+----+----------+
|1 |33 |Zeev|Manilovich|
+--+---+----+----------+

SELECT *
FROM cars;

+--+----------+------+-----------+-------+
|id|model     |color |engine_size|user_id|
+--+----------+------+-----------+-------+
|1 |volkswagen|blue  |1400       |1      |
|2 |delorean  |silver|9999       |1      |
+--+----------+------+-----------+-------+
```

### 同步数据库变更

为了保持数据库同步，我们需要`entimport`能够在数据库变更后更新模式。以下是具体操作。

执行以下SQL代码，为`users`表添加带唯一索引的`phone`列：

```sql
alter table users
    add phone varchar(255) null;

create unique index users_phone_uindex
    on users (phone);
```

表结构应变为：

```sql
describe users;
+-----------+--------------+------+-----+---------+----------------+
| Field     | Type         | Null | Key | Default | Extra          |
+-----------+--------------+------+-----+---------+----------------+
| id        | bigint       | NO   | PRI | NULL    | auto_increment |
| age       | bigint       | NO   |     | NULL    |                |
| name      | varchar(255) | NO   |     | NULL    |                |
| last_name | varchar(255) | YES  |     | NULL    |                |
| phone     | varchar(255) | YES  | UNI | NULL    |                |
+-----------+--------------+------+-----+---------+----------------+
```

现在让我们再次运行 `entimport` 从数据库中获取最新架构：

```shell
go run -mod=mod ariga.io/entimport/cmd/entimport -dialect mysql -dsn "root:pass@tcp(localhost:3306)/entimport"
```

可以看到 `user.go` 文件已被修改：

```go title="entimport-example/ent/schema/user.go"
func (User) Fields() []ent.Field {
	return []ent.Field{field.Int("id"), ..., field.String("phone").Optional().Unique()}
}
```

现在我们可以重新运行 `go generate ./ent` 并使用新架构为 User 实体添加 `phone` 字段。

## 未来计划

如上所述，初始版本支持 MySQL 和 PostgreSQL 数据库，并兼容所有类型的 SQL 关系。我们计划进一步升级工具，添加缺失的 PostgreSQL 字段、默认值等功能。

## 总结

本文介绍了 Ent 社区期待已久的 `entimport` 工具，并通过示例演示了如何将其与 Ent 结合使用。该工具是 Ent 架构导入工具集的又一力作，旨在简化集成流程。如需讨论或支持，请[提交 issue](https://github.com/ariga/entimport/issues/new)。完整示例可[在此获取](https://github.com/zeevmoney/entimport-example)。希望本文对您有所帮助！

:::note[获取更多 Ent 资讯：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 关注 [Twitter](https://twitter.com/entgo_io)
- 加入 [Gophers Slack](https://entgo.io/docs/slack) 的 #ent 频道
- 参与 [Ent Discord 社区](https://discord.gg/qZmPgTE6RX)

:::