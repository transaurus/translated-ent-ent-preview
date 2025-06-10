---
title: Custom migrations
id: custom-migrations
---

:::info[支持仓库]

本节描述的变更可在支持仓库的
[PR #7](https://github.com/rotemtam/ent-versioned-migrations-demo/pull/7/files)
中查看。

:::

## 自定义迁移

某些情况下，您可能需要编写Atlas无法自动生成的定制化迁移脚本。这适用于以下场景：
- 需要对数据库执行Ent当前不支持的操作
- 需要预填充数据库种子数据

本节将演示如何为项目添加自定义迁移。假设我们需要向users表预填充数据作为示例。

## 创建自定义迁移

首先在项目中新建迁移文件：

```shell
atlas migrate new seed_users --dir file://ent/migrate/migrations
```

此时会在`ent/migrate/migrations`目录下生成名为`20221115102552_seed_users.sql`的新文件。

编辑该文件并添加以下SQL语句：

```sql
INSERT INTO `users` (`name`, `email`, `title`)
VALUES ('Jerry Seinfeld', 'jerry@seinfeld.io', 'Mr.'),
       ('George Costanza', 'george@costanza.io', 'Mr.')
```

## 重新计算校验文件

尝试运行新建的自定义迁移：

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

Atlas通过[迁移目录完整性](https://atlasgo.io/concepts/migration-directory-integrity)机制确保线性迁移历史。该机制能有效避免多开发者并行开发时产生的迁移历史冲突。

执行以下命令重新生成迁移目录哈希值以修复错误：

```shell
atlas migrate hash --dir file://ent/migrate/migrations
```

再次运行`atlas migrate apply`，迁移将被成功应用：

```text
atlas migrate apply --dir file://ent/migrate/migrations --url mysql://root:pass@localhost:3306/db
```

Atlas输出如下：

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

下一节我们将学习如何使用Atlas的[代码检查](https://atlasgo.io/versioned/lint)功能自动验证迁移安全性。