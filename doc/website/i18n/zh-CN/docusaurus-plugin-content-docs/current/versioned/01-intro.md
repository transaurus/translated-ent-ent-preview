---
id: intro
title: Introduction
---

## 模式迁移流程

Ent支持两种不同的模式变更管理工作流：

* 自动迁移 - 一种完全在运行时进行的声明式模式迁移。该流程中，Ent会计算已连接数据库与满足`ent.Schema`定义所需数据库模式之间的差异，然后将变更应用到数据库。
* 版本化迁移 - 一种需要预先编写SQL迁移文件，并通过专用工具（如[Atlas](https://atlasgo.io)或[golang-migrate](https://github.com/golang-migrate/migrate)）应用到数据库的工作流。

许多用户最初选择自动迁移流程因其上手简单，但随着项目规模增长，当需要更多迁移过程控制权时，往往会转向版本化迁移流程。

本教程将引导您完成将现有项目从自动迁移升级到版本化迁移的全过程。

## 配套代码库

本教程演示的所有步骤都可以在GitHub上的[rotemtam/ent-versioned-migrations-demo](https://github.com/rotemtam/ent-versioned-migrations-demo)仓库找到。每个章节都会链接到仓库中的相关提交。

我们将要升级的初始Ent项目可在[此处](https://github.com/rotemtam/ent-versioned-migrations-demo/tree/start)查看。

## 自动迁移

本教程假设您已有一个使用自动迁移的现有Ent项目。许多简单项目会在其`main.go`文件中包含类似以下代码块：

```go
package main

func main() {
	// Connect to the database (MySQL for example).
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// Run migration.
	// highlight-next-line
	if err := client.Schema.Create(ctx); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
	// ... Continue with server start.
}
```

这段代码连接数据库后，运行自动迁移工具来创建所有模式资源。

接下来，我们将了解如何为版本化迁移配置项目。