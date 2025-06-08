---
title: Announcing the Upsert API in v0.9.0
author: Rotem Tamir
authorURL: "https://github.com/rotemtam"
authorImageURL: "https://s.gravatar.com/avatar/36b3739951a27d2e37251867b7d44b1a?s=80"
authorTwitter: _rtam
---

距离我们[上一个版本](https://github.com/ent/ent/releases/tag/v0.8.0)发布已近4个月，这是有充分理由的。
今天发布的[0.9.0版本](https://github.com/ent/ent/releases/tag/v0.9.0)包含了多项备受期待的功能，其中最引人注目的是一项已讨论[长达一年半之久](https://github.com/ent/ent/issues/139)且在[Ent用户调查](https://forms.gle/7VZSPVc7D1iu75GV9)中高频请求的功能——Upsert API！

0.9.0版本通过[新特性标志](https://entgo.io/docs/feature-flags#upsert)`sql/upsert`支持"Upsert"式语句。Ent提供了一系列[特性标志](https://entgo.io/docs/feature-flags)，可选择性启用以增强生成代码的功能。这既作为非必需功能的准入机制，也作为未来可能成为核心功能的实验性特性验证手段。

本文将介绍该新特性的应用场景、实用价值及具体使用方法。

### Upsert机制

"Upsert"是数据系统中的常用术语，由"update"和"insert"组合而成，指代当插入记录违反唯一性约束（如ID已存在）时自动转为更新操作的语句。虽然主流关系型数据库没有专门的`UPSERT`语句，但大多支持类似功能的实现方式。

例如，假设我们在SQLite数据库中有如下表定义：

```sql
CREATE TABLE users (
   id integer PRIMARY KEY AUTOINCREMENT,
   email varchar(255) UNIQUE,
   name varchar(255)
)
```

若尝试重复执行相同插入：

```sql
INSERT INTO users (email, name) VALUES ('rotem@entgo.io', 'Rotem Tamir');
INSERT INTO users (email, name) VALUES ('rotem@entgo.io', 'Rotem Tamir');
```

将产生错误：

```
[2021-08-05 06:49:22] UNIQUE constraint failed: users.email
```

多数场景下，我们需要写操作具备[幂等性](https://en.wikipedia.org/wiki/Idempotence)——即多次执行仍保持系统状态一致。

另一些场景中，先行查询记录存在性并非理想方案。对此类需求，SQLite支持在`INSERT`语句中使用[`ON CONFLICT`子句](https://www.sqlite.org/lang_upsert.html)。若需用新值覆盖现有记录可执行：

```sql
INSERT INTO users (email, name) values ('rotem@entgo.io', 'Tamir, Rotem')
ON CONFLICT (email) DO UPDATE SET email=excluded.email, name=excluded.name;
```

若需保留原值，可使用`DO NOTHING`冲突处理：

```sql
INSERT INTO users (email, name) values ('rotem@entgo.io', 'Tamir, Rotem') 
ON CONFLICT DO NOTHING;
```

有时我们需要合并新旧版本，此时可通过特殊形式的`DO UPDATE`实现类似操作：

```sql
INSERT INTO users (email, full_name) values ('rotem@entgo.io', 'Tamir, Rotem') 
ON CONFLICT (email) DO UPDATE SET name=excluded.name ||  ' (formerly: ' || users.name || ')'
```

此例中第二次`INSERT`后，`name`列的值将变为`Tamir, Rotem (formerly: Rotem Tamir)`。虽非实用案例，但足以展示该机制的灵活性。

### Ent中的Upsert实现

假设已有Ent项目包含类似前文`users`表的实体：

```go
// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("email").
			Unique(),
		field.String("name"),
	}
}
```

由于Upsert API是新发布功能，请先通过以下命令更新`ent`版本：

```bash
go get -u entgo.io/ent@v0.9.0
```

接着在代码生成标志中添加`sql/upsert`特性标志，修改`ent/generate.go`文件：

```go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate --feature sql/upsert ./schema
```

最后重新执行项目代码生成：

```go
go generate ./...
```

注意`ent/user_create.go`文件中新增了一个名为`OnConflict`的方法：

```go
// OnConflict allows configuring the `ON CONFLICT` / `ON DUPLICATE KEY` clause
// of the `INSERT` statement. For example:
//
//	client.User.Create().
//		SetEmailAddress(v).
//		OnConflict(
//			// Update the row with the new values
//			// the was proposed for insertion.
//			sql.ResolveWithNewValues(),
//		).
//		// Override some of the fields with custom
//		// update values.
//		Update(func(u *ent.UserUpsert) {
//			SetEmailAddress(v+v)
//		}).
//		Exec(ctx)
//
func (uc *UserCreate) OnConflict(opts ...sql.ConflictOption) *UserUpsertOne {
	uc.conflict = opts
	return &UserUpsertOne{
		create: uc,
	}
}
```

该方法（连同其他新生成的代码）将帮助我们为`User`实体实现upsert行为。首先让我们编写一个测试来复现唯一性约束错误：

```go
func TestUniqueConstraintFails(t *testing.T) {
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	ctx := context.TODO()

	// Create the user for the first time.
	client.User.
		Create().
		SetEmail("rotem@entgo.io").
		SetName("Rotem Tamir").
		SaveX(ctx)

	// Try to create a user with the same email the second time.
	_, err := client.User.
		Create().
		SetEmail("rotem@entgo.io").
		SetName("Rotem Tamir").
		Save(ctx)

	if !ent.IsConstraintError(err) {
		log.Fatalf("expected second created to fail with constraint error")
	}
	log.Printf("second query failed with: %v", err)
}
```

测试通过：

```bash
=== RUN   TestUniqueConstraintFails
2021/08/05 07:12:11 second query failed with: ent: constraint failed: insert node to table "users": UNIQUE constraint failed: users.email
--- PASS: TestUniqueConstraintFails (0.00s)
```

接下来演示如何配置Ent在冲突发生时用新值覆盖现有记录：

```go
func TestUpsertReplace(t *testing.T) {
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	ctx := context.TODO()

	// Create the user for the first time.
	orig := client.User.
		Create().
		SetEmail("rotem@entgo.io").
		SetName("Rotem Tamir").
		SaveX(ctx)

	// Try to create a user with the same email the second time.
	// This time we set ON CONFLICT behavior, and use the `UpdateNewValues`
	// modifier.
	newID := client.User.Create().
		SetEmail("rotem@entgo.io").
		SetName("Tamir, Rotem").
		OnConflict().
		UpdateNewValues().
		// we use the IDX method to receive the ID
		// of the created/updated entity
		IDX(ctx)

	// We expect the ID of the originally created user to be the same as
	// the one that was just updated.
	if orig.ID != newID {
		log.Fatalf("expected upsert to update an existing record")
	}

	current := client.User.GetX(ctx, orig.ID)
	if current.Name != "Tamir, Rotem" {
		log.Fatalf("expected upsert to replace with the new values")
	}
}
```

运行测试：

```bash
=== RUN   TestUpsertReplace
--- PASS: TestUpsertReplace (0.00s)
```

我们也可以使用`Ignore`修饰符让Ent在冲突时保留原记录。测试示例如下：

```go
func TestUpsertIgnore(t *testing.T) {
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	ctx := context.TODO()

	// Create the user for the first time.
	orig := client.User.
		Create().
		SetEmail("rotem@entgo.io").
		SetName("Rotem Tamir").
		SaveX(ctx)

	// Try to create a user with the same email the second time.
	// This time we set ON CONFLICT behavior, and use the `Ignore`
	// modifier.
	client.User.
		Create().
		SetEmail("rotem@entgo.io").
		SetName("Tamir, Rotem").
		OnConflict().
		Ignore().
		ExecX(ctx)

	current := client.User.GetX(ctx, orig.ID)
	if current.FullName != orig.FullName {
		log.Fatalf("expected upsert to keep the original version")
	}
}
```

更多细节请参阅[功能标志](https://entgo.io/docs/feature-flags#upsert)或[Upsert API](https://entgo.io/docs/crud#upsert-one)文档。

### 总结

本文介绍了通过功能标志在Ent v0.9.0中提供的Upsert API——这个期待已久的功能。我们探讨了upsert在应用中的常见场景及其在关系型数据库中的实现原理，最后通过简单示例演示了如何在Ent中使用该API。

若有疑问或需要帮助？欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack/)。

:::note[获取更多Ent资讯：]

- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 加入[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::