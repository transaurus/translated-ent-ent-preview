---
title: Versioned Migrations Management and Migration Directory Integrity
author: Jannik Clausen (MasseElch)
authorURL: "https://github.com/masseelch"
authorImageURL: "https://avatars.githubusercontent.com/u/12862103?v=4"
image: "https://entgo.io/images/assets/migrate/atlas-validate.png"
---

五周前，我们发布了Ent期待已久的数据库变更管理功能：**版本化迁移**。在[发布公告博客](2022-03-14-announcing-versioned-migrations.md)中，我们简要介绍了声明式和基于变更的两种数据库模式同步方案，分析其局限性，并阐释为何[Atlas](https://atlasgo.io)（Ent的底层迁移引擎）尝试融合两者优势的创新工作流值得尝试。我们称之为**版本化迁移编排**，若您尚未阅读该文，现在正是最佳时机！

通过版本化迁移编排，最终生成的迁移文件仍保持"基于变更"的特性，但经由Atlas引擎安全规划。这意味着您依然可以使用熟悉的迁移管理工具，例如[Flyway](https://flywaydb.org/)、[Liquibase](https://liquibase.org/)、[golang-migrate/migrate](https://github.com/golang-migrate/migrate)或[pressly/goose](https://github.com/pressly/goose)来管理Ent服务的数据库变更。

本篇博客将介绍Atlas项目的另一项新功能——**迁移目录完整性文件**（现已在Ent中支持），以及如何将其与您惯用的迁移管理工具结合使用。

### 问题背景

使用版本化迁移时，开发者需警惕以下操作以避免破坏数据库：

1. 修改已执行的迁移历史记录
2. 意外调整迁移文件执行顺序
3. 提交存在语义错误的SQL脚本
理论上代码审查应能拦截这些问题，但实践中许多错误仍可能逃过人工审查，因此自动化预防机制更为可靠。

首个问题（修改历史）多数管理工具通过保存已应用迁移文件的哈希值到数据库进行校验。若哈希不匹配则中止迁移。但该机制在开发周期后期（部署阶段）才生效，如能更早发现问题将节省大量时间和资源。

针对第二和第三个问题，请看以下场景：

![atlas-versioned-migrations-no-conflict](https://entgo.io/images/assets/migrate/no-conflict-2.svg)

该示意图展示了两个未被检测到的错误。首先是迁移文件顺序问题。

团队A与团队B几乎同时从主分支创建特性分支。团队B生成时间戳为**x**的迁移文件后继续开发。团队A在较晚时间生成迁移文件，因此获得时间戳**x+1**。团队A完成特性并合并至主分支，可能已自动部署包含版本**x+1**迁移的生产环境。目前尚无问题。

当团队B合并其包含更早版本**x**迁移的特性时，若代码审查未发现时序问题，该迁移文件将进入生产环境。此时具体行为取决于迁移管理工具的实现逻辑。

多数工具各有解决方案，例如`pressly/goose`采用他们称为[混合版本控制](https://github.com/pressly/goose/issues/63#issuecomment-428681694)的方案。在介绍Atlas（Ent）的独特解决方案前，让我们先快速了解第三个问题：

如果团队A和团队B同时开发需要新增表或列的功能，且使用了相同的名称（例如`users`），他们可能会生成创建该表的语句。先合并代码的团队能成功执行迁移，而后合并的团队则会因表或列已存在导致迁移失败。

### 解决方案

Atlas采用独特方式处理上述问题，其核心目标是尽早发现问题。我们认为最佳介入时机是在版本控制和持续集成（CI）环节。为此，Atlas引入了名为**迁移目录完整性文件**的新机制——即一个与迁移文件共存的`atlas.sum`文件。该文件格式借鉴Go模块的`go.sum`，其内容示例如下：

```text
h1:KRFsSi68ZOarsQAJZ1mfSiMSkIOZlMq4RzyF//Pwf8A=
20220318104614_team_A.sql h1:EGknG5Y6GQYrc4W8e/r3S61Aqx2p+NmQyVz/2m8ZNwA=
```

`atlas.sum`文件首行为整个目录的校验和，后续各行是各迁移文件的哈希值（采用反向单分支Merkle哈希树实现）。通过该文件，我们能在版本控制和CI流程中检测前文所述问题。其核心价值在于：当两个团队同时添加迁移时，系统会立即提示需要人工检查合并冲突。

:::note
跟随以下步骤快速创建示例环境：
1. 创建Go模块并下载依赖
2. 构建基础User模型
3. 启用版本化迁移功能
4. 执行代码生成
5. 启动MySQL容器（使用`docker stop atlas-sum`停止）

```shell
mkdir ent-sum-file
cd ent-sum-file
go mod init ent-sum-file
go install entgo.io/ent/cmd/ent@master
go run entgo.io/ent/cmd/ent new User
sed -i -E 's|^//go(.*)$|//go\1 --feature sql/versioned-migration|' ent/generate.go
go generate ./...
docker run --rm --name atlas-sum --detach --env MYSQL_ROOT_PASSWORD=pass --env MYSQL_DATABASE=ent -p 3306:3306 mysql
```
:::

首先需通过`schema.WithSumFile()`选项启用迁移引擎的`atlas.sum`管理功能。以下示例使用[实例化Ent客户端](/docs/versioned-migrations#from-client)生成迁移文件：

```go
package main

import (
	"context"
	"log"
	"os"

	"ent-sum-file/ent"

	"ariga.io/atlas/sql/migrate"
	"entgo.io/ent/dialect/sql/schema"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/ent")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// Create a local migration directory.
	dir, err := migrate.NewLocalDir("migrations")
	if err != nil {
		log.Fatalf("failed creating atlas migration directory: %v", err)
	}
	// Write migration diff.
	// highlight-start
	err = client.Schema.NamedDiff(ctx, os.Args[1], schema.WithDir(dir), schema.WithSumFile())
	// highlight-end
	if err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
}
```

执行上述命令后，迁移目录中会生成兼容`golang-migrate/migrate`的迁移文件及包含如下内容的`atlas.sum`文件：

```shell
mkdir migrations
go run -mod=mod main.go initial
```

```sql title="20220504114411_initial.up.sql"
-- create "users" table
CREATE TABLE `users` (`id` bigint NOT NULL AUTO_INCREMENT, PRIMARY KEY (`id`)) CHARSET utf8mb4 COLLATE utf8mb4_bin;

```

```sql title="20220504114411_initial.down.sql"
-- reverse: create "users" table
DROP TABLE `users`;

```

```text title="atlas.sum"
h1:SxbWjP6gufiBpBjOVtFXgXy7q3pq1X11XYUxvT4ErxM=
20220504114411_initial.down.sql h1:OllnelRaqecTrPbd2YpDbBEymCpY/l6ihbyd/tVDgeY=
20220504114411_initial.up.sql h1:o/6yOczGSNYQLlvALEU9lK2/L6/ws65FrHJkEk/tjBk=
```

可见`atlas.sum`为每个迁移文件记录了哈希值。启用该功能后，当团队A和团队B各自生成迁移文件时都会产生此文件。此时版本控制系统会在第二个团队尝试合并时触发冲突告警。

![atlas-versioned-migrations-no-conflict](https://entgo.io/images/assets/migrate/conflict-2.svg)

:::note
后续步骤中我们通过`go run -mod=mod ariga.io/atlas/cmd/atlas`调用Atlas CLI，您也可以参照[安装指南](https://atlasgo.io/cli/getting-started/setting-up#install-the-cli)全局安装后直接使用`atlas`命令。
:::

可随时运行以下命令验证`atlas.sum`与迁移目录的同步状态（当前应无错误输出）：

```shell
go run -mod=mod ariga.io/atlas/cmd/atlas migrate validate
```

然而，如果您手动修改了迁移文件（例如添加新的SQL语句、编辑现有语句或创建全新文件），`atlas.sum`文件将与迁移目录内容失去同步。此时尝试为模式变更生成新迁移文件会被Atlas迁移引擎阻止。您可以通过创建新的空迁移文件并再次运行`main.go`来验证：

```shell
go run -mod=mod ariga.io/atlas/cmd/atlas migrate new migrations/manual_version.sql --format golang-migrate
go run -mod=mod main.go initial
# 2022/05/04 15:08:09 failed creating schema resources: validating migration directory: checksum mismatch
# exit status 1

```

执行`atlas migrate validate`命令会显示相同错误：

```shell
go run -mod=mod ariga.io/atlas/cmd/atlas migrate validate
# Error: checksum mismatch
# 
# You have a checksum error in your migration directory.
# This happens if you manually create or edit a migration file.
# Please check your migration files and run
# 
# 'atlas migrate hash --force'
# 
# to re-hash the contents and resolve the error.
# 
# exit status 1
```

要使`atlas.sum`文件重新与迁移目录同步，我们可以再次使用Atlas CLI：

```shell
go run -mod=mod ariga.io/atlas/cmd/atlas migrate hash --force
```

作为安全措施，Atlas CLI不会操作与`atlas.sum`文件不同步的迁移目录。因此您需要在命令中添加`--force`标志。

针对开发者忘记在手动修改后更新`atlas.sum`文件的情况，您可以在CI流程中添加`atlas migrate validate`检查。我们正在开发开箱即用的GitHub Action和CI解决方案，该方案将自动执行此操作（以及其他功能）。

### 总结

本文简要介绍了基于变更的SQL文件工作时常见的模式迁移问题，并提出了基于Atlas项目的解决方案来增强迁移安全性。

有问题需要帮助？欢迎加入我们的[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)。

:::note[获取更多Ent资讯：]

- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)
:::