---
title: Database Locking Techniques with Ent
author: Rotem Tamir
authorURL: "https://github.com/rotemtam"
authorImageURL: "https://s.gravatar.com/avatar/36b3739951a27d2e37251867b7d44b1a?s=80"
authorTwitter: _rtam
---

锁是任何并发计算机程序的基础构建模块之一。当多个操作同时发生时，程序员会借助锁来确保对资源的并发访问具有互斥性。锁（以及其他互斥原语）存在于技术栈的各个层面，从底层的CPU指令到应用层API（如Go语言中的`sync.Mutex`）。

在使用关系型数据库时，应用开发者的常见需求之一是对记录加锁。假设有一个电商网站的`inventory`库存表，其中包含表示商品状态的`state`列，可能值为`available`（可售）或`purchased`（已售）。为避免两个用户误认为成功购买了同一商品，应用程序必须防止两个操作同时将商品状态从可售改为已售。

如何实现这种保证？仅让服务器在更新前检查商品是否`available`是不够的。设想两个用户同时购买同一商品：两个请求会几乎同时到达应用服务器，两者都会查询数据库获取商品状态并看到`available`状态。此时两个请求处理器都会发出`UPDATE`查询将状态改为`purchased`并将`buyer_id`设为当前用户ID。虽然两个更新都会成功执行，但最终记录状态将由最后执行`UPDATE`的用户获得购买权。

多年来，开发者逐渐形成了多种技术来实现这种业务保证。有些依赖于数据库提供的显式锁机制，有些则利用数据库ACID特性实现互斥。本文将探讨如何通过Ent框架实现其中两种技术。

### 乐观锁

乐观锁（有时称为乐观并发控制）是一种无需显式获取记录锁的加锁技术。

其核心工作原理如下：

- 每条记录分配一个数值版本号（必须单调递增，通常使用最近更新的Unix时间戳）
- 事务读取记录时获取当前版本号
- 执行`UPDATE`语句时：
    - 语句必须包含版本号未变更的条件（例如`WHERE id=<id> AND version=<previous version>`）
    - 必须递增版本号（可+1或设为当前时间戳）
- 数据库返回受影响行数。若为0则表示读取记录后、更新前版本号已变更，事务失败需回滚并重试

乐观锁通常适用于"低竞争"环境（事务冲突概率较低），且要求所有写操作都必须遵守版本控制逻辑。如果存在不遵守该逻辑的写入操作，此技术将失效。

下面演示如何通过Ent实现该技术：

首先定义`User`的`ent.Schema`，包含表示在线状态的`online`布尔字段和记录版本号的`int64`字段：

```go
// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Bool("online"),
		field.Int64("version").
			DefaultFunc(func() int64 {
				return time.Now().UnixNano()
			}).
			Comment("Unix time of when the latest update occurred")
	}
}
```

接着实现基于乐观锁的`online`字段更新逻辑：

```go
func optimisticUpdate(tx *ent.Tx, prev *ent.User, online bool) error {
	// The next version number for the record must monotonically increase
	// using the current timestamp is a common technique to achieve this. 
	nextVer := time.Now().UnixNano()

	// We begin the update operation:
	n := tx.User.Update().

		// We limit our update to only work on the correct record and version:
		Where(user.ID(prev.ID), user.Version(prev.Version)).

		// We set the next version:
		SetVersion(nextVer).

		// We set the value we were passed by the user:
		SetOnline(online).
		SaveX(context.Background())

	// SaveX returns the number of affected records. If this value is 
	// different from 1 the record must have been changed by another
	// process.
	if n != 1 {
		return fmt.Errorf("update failed: user id=%d updated by another process", prev.ID)
	}
	return nil
}
```

接下来，我们编写一个测试来验证当两个进程尝试编辑同一条记录时，只有其中一个会成功：

```go
func TestOCC(t *testing.T) {
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	ctx := context.Background()

	// Create the user for the first time.
	orig := client.User.Create().SetOnline(true).SaveX(ctx)

	// Read another copy of the same user.
	userCopy := client.User.GetX(ctx, orig.ID)

	// Open a new transaction:
	tx, err := client.Tx(ctx)
	if err != nil {
		log.Fatalf("failed creating transaction: %v", err)
	}

	// Try to update the record once. This should succeed.
	if err := optimisticUpdate(tx, userCopy, false); err != nil {
		tx.Rollback()
		log.Fatal("unexpected failure:", err)
	}

	// Try to update the record a second time. This should fail.
	err = optimisticUpdate(tx, orig, false)
	if err == nil {
		log.Fatal("expected second update to fail")
	}
	fmt.Println(err)
}
```

运行测试：

```go
=== RUN   TestOCC
update failed: user id=1 updated by another process
--- PASS: Test (0.00s)
```

太棒了！通过乐观锁机制，我们可以避免两个进程相互干扰！

### 悲观锁

如前所述，乐观锁并非适用于所有场景。对于更倾向于将锁完整性维护职责委托给数据库的用例，某些数据库引擎（如MySQL、Postgres和MariaDB，但不包括SQLite）提供了悲观锁功能。这些数据库支持在`SELECT`语句中添加`SELECT ... FOR UPDATE`修饰符。MySQL文档对此的解释是：

> `SELECT ... FOR UPDATE`会读取最新的可用数据，并对读取的每行设置排他锁。因此，它会在这些行上设置与搜索型SQL `UPDATE`语句相同的锁。

用户也可以使用`SELECT ... FOR SHARE`语句，文档中说明该语句：

> 对读取的所有行设置共享模式锁。其他会话可以读取这些行，但在您的事务提交前无法修改它们。如果这些行被其他尚未提交的事务修改过，您的查询将等待该事务结束后使用最新值。

Ent近期通过名为`sql/lock`的特性标志增加了对`FOR SHARE`/`FOR UPDATE`语句的支持。要启用该功能，需修改`generate.go`文件添加`--feature sql/lock`参数：

```go
//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate --feature sql/lock ./schema 
```

接下来，我们实现一个使用悲观锁确保只有一个进程能更新`User`对象`online`字段的函数：

```go
func pessimisticUpdate(tx *ent.Tx, id int, online bool) (*ent.User, error) {
	ctx := context.Background()

	// On our active transaction, we begin a query against the user table
	u, err := tx.User.Query().

		// We add a predicate limiting the lock to the user we want to update.
		Where(user.ID(id)).

		// We use the ForUpdate method to tell ent to ask our DB to lock
		// the returned records for update.
		ForUpdate(
			// We specify that the query should not wait for the lock to be
			// released and instead fail immediately if the record is locked.
			sql.WithLockAction(sql.NoWait),
		).
		Only(ctx)
	
	// If we failed to acquire the lock we do not proceed to update the record.
	if err != nil {
		return nil, err
	}
	
	// Finally, we set the online field to the desired value. 
	return u.Update().SetOnline(online).Save(ctx)
}
```

现在编写测试验证当两个进程尝试编辑同一条记录时，只有其中一个会成功：

```go
func TestPessimistic(t *testing.T) {
	ctx := context.Background()
	client := enttest.Open(t, dialect.MySQL, "root:pass@tcp(localhost:3306)/test?parseTime=True")

	// Create the user for the first time.
	orig := client.User.Create().SetOnline(true).SaveX(ctx)

	// Open a new transaction. This transaction will acquire the lock on our user record.
	tx, err := client.Tx(ctx)
	if err != nil {
		log.Fatalf("failed creating transaction: %v", err)
	}
	defer tx.Commit()
	
	// Open a second transaction. This transaction is expected to fail at 
	// acquiring the lock on our user record. 
	tx2, err := client.Tx(ctx)
	if err != nil {
		log.Fatalf("failed creating transaction: %v", err)
	}
	defer tx.Commit()
	
	// The first update is expected to succeed.
	if _, err := pessimisticUpdate(tx, orig.ID, true); err != nil {
		log.Fatalf("unexpected error: %s", err)
	}
	
	// Because we did not run tx.Commit yet, the row is still locked when
	// we try to update it a second time. This operation is expected to 
	// fail. 
	_, err = pessimisticUpdate(tx2, orig.ID, true)
	if err == nil {
		log.Fatal("expected second update to fail")
	}
	fmt.Println(err)
}
```

这个示例中有几点值得注意：

- 注意我们使用真实的MySQL实例运行测试，因为SQLite不支持`SELECT .. FOR UPDATE`
- 为简化示例，我们使用`sql.NoWait`选项让数据库在无法获取锁时立即返回错误。这意味着调用应用在收到错误后需要重试写入操作。如果不指定该选项，可以创建阻塞流程直到锁释放后继续执行（无需重试）。虽然这种设计在某些场景下很有用，但并不总是理想选择
- 必须始终提交事务。忘记提交可能导致严重问题——记住在锁保持期间，其他任何操作都无法读取或更新该记录

运行测试：

```go
=== RUN   TestPessimistic
Error 3572: Statement aborted because lock(s) could not be acquired immediately and NOWAIT is set.
--- PASS: TestPessimistic (0.08s)
```

成功！我们利用MySQL的"锁定读取"功能及Ent的新支持，实现了提供真正互斥保证的锁机制。

### 结论

本文首先介绍了促使开发者在数据库操作中使用锁技术的业务需求场景，随后阐述了实现数据库记录更新互斥的两种不同方案，并演示了如何通过Ent应用这些技术。

有任何疑问或需要入门帮助？欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack)。

:::note[获取更多 Ent 资讯与更新：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 在[Twitter](https://twitter.com/entgo_io)上关注我们
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 加入[Ent Discord 服务器](https://discord.gg/qZmPgTE6RX)

:::