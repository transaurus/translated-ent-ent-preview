---
id: code-gen
title: Introduction
---

## 安装

项目自带名为`ent`的代码生成工具。执行以下命令安装`ent`：

```bash
go get entgo.io/ent/cmd/ent
```

## 初始化新Schema

执行`ent init`命令生成一个或多个Schema模板：

```bash
go run -mod=mod entgo.io/ent/cmd/ent new User Pet
```

`init`命令将在`ent/schema`目录下创建两个Schema文件（`user.go`和`pet.go`）。若`ent`目录不存在，工具会自动创建。项目约定在根目录下建立`ent`目录。

## 生成资源文件

添加若干[字段](schema-fields.mdx)和[边](schema-edges.mdx)后，执行`ent generate`生成实体操作资源文件。可在项目根目录运行该命令，或使用`go generate`：

```bash
go generate ./ent
```

`generate`命令会为Schema生成以下资源：

- 用于操作图的`Client`和`Tx`对象
- 每个Schema类型的CRUD构建器（参见[CRUD](crud.mdx)）
- 各Schema类型对应的实体对象（Go结构体）
- 包含构建器使用的常量和谓词的包
- 支持SQL方言的`migrate`包（参见[迁移](migrate.md)）
- 用于添加变更中间件的`hook`包（参见[钩子](hooks.md)）

## `entc`与`ent`的版本兼容性

使用`ent`CLI时，需确保CLI版本与项目中引用的`ent`版本**完全一致**。

可通过`go generate`命令强制使用`go.mod`文件中指定的版本。若项目未使用[Go模块](https://github.com/golang/go/wiki/Modules#quick-start)，请先初始化：

```console
go mod init <project>
```

然后重新执行以下命令将`ent`添加至`go.mod`文件：

```console
go get entgo.io/ent/cmd/ent
```

在`<project>/ent`目录下创建`generate.go`文件：

```go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
```

最后在项目根目录执行`go generate ./ent`即可运行Schema代码生成。

## 代码生成选项

执行`ent generate -h`查看所有代码生成选项：

```console
generate go code for the schema directory

Usage:
  ent generate [flags] path

Examples:
  ent generate ./ent/schema
  ent generate github.com/a8m/x

Flags:
      --feature strings       extend codegen with additional features
      --header string         override codegen header
  -h, --help                  help for generate
      --storage string        storage driver to support in codegen (default "sql")
      --target string         target directory for codegen
      --template strings      external templates to execute
```

## 存储选项

`ent`支持生成SQL和Gremlin两种方言的资源文件，默认生成SQL方言。

## 外部模板

`ent`支持执行外部Go模板。若模板名与内置模板重复将覆盖原模板，否则会生成与模板同名的输出文件。支持`file`、`dir`和`glob`三种格式：

```console
go run -mod=mod entgo.io/ent/cmd/ent generate --template <dir-path> --template glob="path/to/*.tmpl" ./ent/schema
```

更多示例详见[外部模板文档](templates.md)。

## 以包形式使用`entc`

另一种代码生成方式是在`ent/entc.go`中创建以下内容，然后通过`ent/generate.go`执行：

```go title="ent/entc.go"
// +build ignore

package main

import (
	"log"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"entgo.io/ent/schema/field"
)

func main() {
	if err := entc.Generate("./schema", &gen.Config{}); err != nil {
		log.Fatal("running ent codegen:", err)
	}
}
```

```go title="ent/generate.go"
package ent

//go:generate go run -mod=mod entc.go
```

完整示例可在 [GitHub](https://github.com/ent/ent/tree/master/examples/entcpkg) 上查看。

## 模式描述

要获取图模式的描述信息，请运行：

```bash
go run -mod=mod entgo.io/ent/cmd/ent describe ./ent/schema
```

输出示例如下：

```console
Pet:
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| Field |  Type   | Unique | Optional | Nillable | Default | UpdateDefault | Immutable |       StructTag       | Validators |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| id    | int     | false  | false    | false    | false   | false         | false     | json:"id,omitempty"   |          0 |
	| name  | string  | false  | false    | false    | false   | false         | false     | json:"name,omitempty" |          0 |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	+-------+------+---------+---------+----------+--------+----------+
	| Edge  | Type | Inverse | BackRef | Relation | Unique | Optional |
	+-------+------+---------+---------+----------+--------+----------+
	| owner | User | true    | pets    | M2O      | true   | true     |
	+-------+------+---------+---------+----------+--------+----------+
	
User:
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| Field |  Type   | Unique | Optional | Nillable | Default | UpdateDefault | Immutable |       StructTag       | Validators |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| id    | int     | false  | false    | false    | false   | false         | false     | json:"id,omitempty"   |          0 |
	| age   | int     | false  | false    | false    | false   | false         | false     | json:"age,omitempty"  |          0 |
	| name  | string  | false  | false    | false    | false   | false         | false     | json:"name,omitempty" |          0 |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	+------+------+---------+---------+----------+--------+----------+
	| Edge | Type | Inverse | BackRef | Relation | Unique | Optional |
	+------+------+---------+---------+----------+--------+----------+
	| pets | Pet  | false   |         | O2M      | false  | true     |
	+------+------+---------+---------+----------+--------+----------+
```

## 代码生成钩子

`entc` 包提供了在代码生成阶段添加钩子（中间件）的功能。该功能适用于为模式添加自定义验证器，或基于图模式生成额外资产。

```go
// +build ignore

package main

import (
	"fmt"
	"log"
	"reflect"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
)

func main() {
	err := entc.Generate("./schema", &gen.Config{
		Hooks: []gen.Hook{
			EnsureStructTag("json"),
		},
	})
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}

// EnsureStructTag ensures all fields in the graph have a specific tag name.
func EnsureStructTag(name string) gen.Hook {
	return func(next gen.Generator) gen.Generator {
		return gen.GenerateFunc(func(g *gen.Graph) error {
			for _, node := range g.Nodes {
				for _, field := range node.Fields {
					tag := reflect.StructTag(field.StructTag)
					if _, ok := tag.Lookup(name); !ok {
						return fmt.Errorf("struct tag %q is missing for field %s.%s", name, node.Name, field.Name)
					}
				}
			}
			return next.Generate(g)
		})
	}
}
```

## 外部依赖

若需扩展 `ent` 包下生成的客户端和构建器，并注入外部依赖作为结构体字段，请在 [`ent/entc.go`](#use-entc-as-a-package) 文件中使用 `entc.Dependency` 选项：

```go title="ent/entc.go" {3-12}
func main() {
	opts := []entc.Option{
		entc.Dependency(
			entc.DependencyType(&http.Client{}),
		),
		entc.Dependency(
			entc.DependencyName("Writer"),
			entc.DependencyTypeInfo(&field.TypeInfo{
				Ident:   "io.Writer",
				PkgPath: "io",
			}),
		),
	}
	if err := entc.Generate("./schema", &gen.Config{}, opts...); err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

然后在应用程序中使用：

```go title="example_test.go" {5-6,15-16}
func Example_Deps() {
	client, err := ent.Open(
		"sqlite3",
		"file:ent?mode=memory&cache=shared&_fk=1",
		ent.Writer(os.Stdout),
		ent.HTTPClient(http.DefaultClient),
	)
	if err != nil {
		log.Fatalf("failed opening connection to sqlite: %v", err)
	}
	defer client.Close()
	// An example for using the injected dependencies in the generated builders.
	client.User.Use(func(next ent.Mutator) ent.Mutator {
		return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
			_ = m.HTTPClient
			_ = m.Writer
			return next.Mutate(ctx, m)
		})
	})
	// ...
}
```

完整示例可在 [GitHub](https://github.com/ent/ent/tree/master/examples/entcpkg) 上查看。

## 功能开关

`entc` 包提供了一系列可通过标志位添加或移除的代码生成特性。

更多信息请参阅 [功能开关页面](features.md)。