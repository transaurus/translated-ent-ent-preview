---
title: Changing a column’s type with zero-downtime using Atlas
author: Ronen Lubin (ronenlu)
authorURL: "https://github.com/ronenlu"
authorImageURL: "https://avatars.githubusercontent.com/u/63970571?v=4"
---

修改数据库表中的列类型看似简单，实则是一项高风险操作，可能导致服务端与数据库的兼容性问题。本文将探讨开发者如何在不造成应用停机的情况下完成此类变更。

近期在为[Ariga Cloud](https://atlasgo.io/cloud/getting-started)开发功能时，我需要将Ent字段类型从非结构化的二进制大对象(blob)改为结构化的JSON字段。这一变更是为了使用[JSON谓词](https://entgo.io/docs/predicates/#json-predicates)来实现更高效的查询。

原始模式在我们的云产品可视化图表中呈现如下：

![教程图1](https://entgo.io/images/assets/migrate-column-type/users_table.png)

由于旧列数据可能无法直接转换为JSON格式，我们不能简单地将数据复制到新列。

过去，人们可以接受先停止服务、迁移数据库模式、再启动兼容新模式的版本。如今业务需求要求更高可用性，工程团队必须实现零停机变更。

根据Martin Fowler在[《进化式数据库设计》](https://www.martinfowler.com/articles/evodb.html)中的定义，满足此类需求的通用模式是采用"过渡阶段"。

> 过渡阶段指"数据库同时支持新旧访问模式的时期，让旧系统有充分时间逐步迁移"，如下图所示：

![教程图2](https://www.martinfowler.com/articles/evodb/stages_refactoring.jpg)
图片来源：martinfowler.com

我们规划了5个向后兼容的变更步骤：

* 创建名为`meta_json`的JSON类型新列
* 部署支持双写的应用版本（新记录同时写入新旧列，读取仍使用旧列）
* 将旧列数据回填至新列
* 部署从新列读取数据的应用版本
* 删除旧列

### 版本化迁移

我们项目使用Ent的[版本化迁移](https://entgo.io/docs/versioned-migrations)工作流管理数据库模式。该方案提供对数据库模式变更的精细控制，非常适合实现我们的计划。若您的项目使用[自动迁移](https://entgo.io/docs/migrate)，请先[升级](https://entgo.io/docs/versioned/intro)至版本化迁移。

:::note
通过[数据迁移](https://entgo.io/docs/data-migrations/#automatic-migrations)功能也可实现类似效果，但本文聚焦版本化迁移方案
:::

### 使用Ent创建JSON列：

首先在用户模式中添加新的JSON类型字段。

``` go title="types/types.go"
type Meta struct {
	CreateTime time.Time `json:"create_time"`
	UpdateTime time.Time `json:"update_time"`
}
```

``` go title="ent/schema/user.go"
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Bytes("meta"),
        field.JSON("meta_json", &types.Meta{}).Optional(),
    }
}
```

接着运行代码生成更新应用模式：

``` shell
go generate ./...
```

然后运行[自动迁移规划](https://entgo.io/docs/versioned/auto-plan)脚本，生成包含数据库升级所需SQL语句的迁移文件集。

``` shell
go run -mod=mod ent/migrate/main.go add_json_meta_column
```

最终生成的变更描述文件：

``` sql
-- modify "users" table
ALTER TABLE `users` ADD COLUMN `meta_json` json NULL;
```

现在，我们将使用[Atlas](https://atlasgo.io)应用创建的迁移文件：

``` shell
atlas migrate apply \
  --dir "file://ent/migrate/migrations"
  --url mysql://root:pass@localhost:3306/ent
```

最终，数据库中生成的模式如下：

![教程图示3](https://entgo.io/images/assets/migrate-column-type/users_table_add_column.png)

### 开始双写数据列

生成JSON类型后，我们将开始向新列写入数据：

``` diff
-		err := client.User.Create().
-			SetMeta(input.Meta).
-			Exec(ctx)
+		var meta types.Meta
+		if err := json.Unmarshal(input.Meta, &meta); err != nil {
+			return nil, err
+		}
+		err := client.User.Create().
+			SetMetaJSON(&meta).
+			Exec(ctx)
```

为确保写入新列`meta_json`的值能同步到旧列，可利用Ent的[Schema Hooks](https://entgo.io/docs/hooks/#schema-hooks)功能。这需要在主程序中添加`ent/runtime`的空白导入以[注册钩子](https://entgo.io/docs/hooks/#hooks-registration)，避免循环引用：

``` go
// Hooks of the User.
func (User) Hooks() []ent.Hook {
	return []ent.Hook{
		hook.On(
			func(next ent.Mutator) ent.Mutator {
				return hook.UserFunc(func(ctx context.Context, m *gen.UserMutation) (ent.Value, error) {
					meta, ok := m.MetaJSON()
					if !ok {
						return next.Mutate(ctx, m)
					}
					if b, err := json.Marshal(meta); err != nil {
						return nil, err
					}
					m.SetMeta(b)
					return next.Mutate(ctx, m)
				})
			},
			ent.OpCreate,
		),
	}
}
```

确认双写机制后即可安全部署至生产环境。

### 回填旧列数据

当前生产数据库中存在两列：一列以blob格式存储meta对象，另一列以JSON格式存储。由于JSON列是新增的，可能存在空值，因此需要用旧列数据进行回填。

为此，我们手动创建SQL迁移文件，将旧blob列的数据填充至新JSON列。

:::note
也可通过[WriteDriver](https://entgo.io/docs/data-migrations#versioned-migrations)编写生成该数据迁移文件的Go代码。
:::

创建新的空迁移文件：

``` shell
atlas migrate new --dir file://ent/migrate/migrations
```

对于users表中JSON值为空的行（即新增列之前添加的行），我们尝试将meta对象解析为有效JSON。若解析成功，则将结果值填入`meta_json`列，否则标记为空值。

下一步编辑迁移文件：

``` sql
UPDATE users
SET meta_json = CASE
        -- when meta is valid json stores it as is.
        WHEN JSON_VALID(cast(meta as char)) = 1 THEN cast(cast(meta as char) as json)
        -- if meta is not valid json, store it as an empty object.
        ELSE JSON_SET('{}')
    END
WHERE meta_json is null;
```

修改迁移文件后重新计算迁移目录哈希：

``` shell
atlas migrate hash --dir "file://ent/mirate/migrations"
```

可通过在本地数据库执行所有前置迁移文件、注入测试数据并应用最新迁移来验证迁移文件是否符合预期。

测试完成后应用迁移文件：

``` shell
atlas migrate apply \
  --dir "file://ent/migrate/migrations"
  --url mysql://root:pass@localhost:3306/ent 
```

此时可再次部署至生产环境。

### 切换读取至新列并删除旧列

现在`meta_json`列已有数据，可将读取操作从旧字段切换至新字段。

无需在每次读取时从`user.meta`解码数据，直接使用`meta_json`字段即可：

``` diff 	
-       var meta types.Meta
-       if err = json.Unmarshal(user.Meta, &meta); err != nil {
-	        return nil, err
-       }
-       if meta.CreateTime.Before(time.Unix(0, 0)) {
-	        return nil, errors.New("invalid create time")
-       }
+       if user.MetaJSON.CreateTime.Before(time.Unix(0, 0)) {
+	        return nil, errors.New("invalid create time")
+       }
```

完成读取切换后部署至生产环境。

### 删除旧列

由于旧列已不再使用，现在可以从Ent模式中移除其字段定义。

``` diff
func (User) Fields() []ent.Field {
    return []ent.Field{
-     field.Bytes("meta"),
      field.JSON("meta_json", &types.Meta{}).Optional(),
    }
}

```

重新生成Ent模式时启用[Drop Column](https://entgo.io/docs/migrate/#drop-resources)功能。

``` shell
go run -mod=mod ent/migrate/main.go drop_user_meta_column
```

至此我们已完成：创建新字段、重定向写入、数据回填及删除旧列——最后只需将代码合并至版本控制系统并部署至生产环境！

### 总结

本文演示了如何通过Atlas版本化迁移与Ent集成，实现生产环境数据库列类型变更的零停机方案。

有问题需要帮助？欢迎加入我们的 [Ent Discord 服务器](https://discord.gg/qZmPgTE6RX)交流讨论。

:::note[获取更多 Ent 资讯与更新：]

- 订阅我们的 [新闻通讯](https://entgo.substack.com/)
- 关注我们的 [Twitter](https://twitter.com/entgo_io)
- 加入 [Gophers Slack](https://entgo.io/docs/slack) 的 #ent 频道
- 加入 [Ent Discord 服务器](https://discord.gg/qZmPgTE6RX)
:::