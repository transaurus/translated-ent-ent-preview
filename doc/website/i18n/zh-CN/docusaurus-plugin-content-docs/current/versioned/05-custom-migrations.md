---
title: Custom migrations
id: custom-migrations
---

:::info[支持仓库]

本节描述的变更可在支持仓库的
[PR #7](https://github.com/rotemtam/ent-versioned-migrations-demo/pull/7/files)
中找到。

:::

## 自定义迁移

某些情况下，您可能需要编写Atlas无法自动生成的自定义迁移。这在需要执行Ent当前不支持的数据库变更，或需要为数据库预置数据时特别有用。

本节我们将学习如何向项目添加自定义迁移。为便于演示，假设我们需要向users表预置一些数据。

## 创建自定义迁移

首先在项目中添加新的迁移文件：

```shell
atlas migrate new seed_users --dir file://ent/migrate/migrations
```

可以看到在`ent/migrate/migrations`目录下创建了名为`20221115102552_seed_users.sql`的新文件。

打开该文件并添加以下SQL语句：

```sql
INSERT INTO `users` (`name`, `email`, `title`)
VALUES ('Jerry Seinfeld', 'jerry@seinfeld.io', 'Mr.'),
       ('George Costanza', 'george@costanza.io', 'Mr.')
```

## 重新计算校验和文件

尝试运行新的自定义迁移：

```shell
atlas migrate apply --dir file://ent/migrate/migrations --url mysql://root:pass@localhost:3306/db
```

Atlas报错如下：

```text
You have a checksum error in your migration directory.
This happens if you manually create or edit a migration file.
Please check your migration files and run

'atlas migrate hash'

to re-hash the contents and resolve the error

Error: checksum mismatch
```

Atlas通过[迁移目录完整性](https://atlasgo.io/concepts/migration-directory-integrity)机制确保线性迁移历史。这样当多名开发者并行工作时，可以保证合并后的迁移历史正确无误。

重新计算迁移目录内容的哈希值以解决该错误：

```shell
atlas migrate hash --dir file://ent/migrate/migrations
```

再次运行`atlas migrate apply`，迁移将成功执行：

```text
atlas migrate apply --dir file://ent/migrate/migrations --url mysql://root:pass@localhost:3306/db
```

Atlas输出：

```text
Migrating to version 20221115102552 from 20221115101649 (1 migrations in total):

  -- migrating version 20221115102552
    -> INSERT INTO `users` (`name`, `email`, `title`)
VALUES ('Jerry Seinfeld', 'jerry@seinfeld.io', 'Mr.'),
       ('George Costanza', 'george@costanza.io', 'Mr.')
  -- ok (9.077102ms)

  -------------------------
  -- 19.857555ms
  -- 1 migrations 
  -- 1 sql statements
```

下一节我们将学习如何使用Atlas的[代码检查](https://atlasgo.io/versioned/lint)功能自动验证模式迁移的安全性。