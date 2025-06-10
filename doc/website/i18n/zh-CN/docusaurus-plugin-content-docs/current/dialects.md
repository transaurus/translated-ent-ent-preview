---
id: dialects
title: Supported Dialects
---

## MySQL

MySQL支持[Migration](migrate.md)章节中提到的所有功能，并持续在以下三个版本进行测试：`5.6.35`、`5.7.26` 和 `8`。

## MariaDB

MariaDB支持[Migration](migrate.md)章节中提到的所有功能，并持续在以下三个版本进行测试：`10.2`、`10.3` 和最新版本。

## PostgreSQL

PostgreSQL支持[Migration](migrate.md)章节中提到的所有功能，并持续在以下五个版本进行测试：`11`、`12`、`13`、`14` 和 `15`。

## CockroachDB **(<ins>预览版</ins>)**

CockroachDB支持目前处于预览阶段，需使用[Atlas迁移引擎](migrate.md#atlas-integration)。  
当前集成测试基于版本`v21.2.11`进行。

## SQLite

通过[Atlas](https://github.com/ariga/atlas)驱动，SQLite支持[Migration](migrate.md)章节中提到的所有功能。需注意部分变更（如列修改）会按照[SQLite官方文档](https://www.sqlite.org/lang_altertable.html#otheralter)描述的流程在临时表上执行操作序列。

## Gremlin

Gremlin不支持迁移和索引功能，**<ins>目前视为实验性功能</ins>**。

## TiDB **(<ins>预览版</ins>)**

TiDB支持目前处于预览阶段，需使用[Atlas迁移引擎](migrate.md#atlas-integration)。  
TiDB与MySQL兼容，因此MySQL支持的功能理论上也适用于TiDB。  
已知兼容性问题列表请访问：https://docs.pingcap.com/tidb/stable/mysql-compatibility  
当前集成测试基于版本`5.4.0`和`6.0.0`进行。