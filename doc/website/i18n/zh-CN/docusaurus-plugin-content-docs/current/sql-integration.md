---
id: sql-integration
title: sql.DB Integration
---

以下示例展示如何将自定义的 `sql.DB` 对象传递给 `ent.Client`。

## 配置 `sql.DB`

第一种方式：

```go
package main

import (
    "time"

    "<your_project>/ent"
    "entgo.io/ent/dialect/sql"
)

func Open() (*ent.Client, error) {
    drv, err := sql.Open("mysql", "<mysql-dsn>")
    if err != nil {
    	return nil, err
    }
    // Get the underlying sql.DB object of the driver.
    db := drv.DB()
    db.SetMaxIdleConns(10)
    db.SetMaxOpenConns(100)
    db.SetConnMaxLifetime(time.Hour)
    return ent.NewClient(ent.Driver(drv)), nil
}
```

第二种方式：

```go
package main

import (
    "database/sql"
    "time"

    "<your_project>/ent"
    entsql "entgo.io/ent/dialect/sql"
)

func Open() (*ent.Client, error) {
    db, err := sql.Open("mysql", "<mysql-dsn>")
    if err != nil {
    	return nil, err
    }
    db.SetMaxIdleConns(10)
    db.SetMaxOpenConns(100)
    db.SetConnMaxLifetime(time.Hour)
    // Create an ent.Driver from `db`.
    drv := entsql.OpenDB("mysql", db)
    return ent.NewClient(ent.Driver(drv)), nil
}
```

## 在 MySQL 中使用 Opencensus

```go
package main

import (
	"context"
	"database/sql"
	"database/sql/driver"

	"<project>/ent"

	"contrib.go.opencensus.io/integrations/ocsql"
	"entgo.io/ent/dialect"
	entsql "entgo.io/ent/dialect/sql"
	"github.com/go-sql-driver/mysql"
)

type connector struct {
	dsn string
}

func (c connector) Connect(context.Context) (driver.Conn, error) {
	return c.Driver().Open(c.dsn)
}

func (connector) Driver() driver.Driver {
	return ocsql.Wrap(
		mysql.MySQLDriver{},
		ocsql.WithAllTraceOptions(),
		ocsql.WithRowsClose(false),
		ocsql.WithRowsNext(false),
		ocsql.WithDisableErrSkip(true),
	)
}

// Open new connection and start stats recorder.
func Open(dsn string) *ent.Client {
	db := sql.OpenDB(connector{dsn})
	// Create an ent.Driver from `db`.
	drv := entsql.OpenDB(dialect.MySQL, db)
	return ent.NewClient(ent.Driver(drv))
}
```

## 在 PostgreSQL 中使用 pgx

```go
package main

import (
	"context"
	"database/sql"
	"log"

	"<project>/ent"

	"entgo.io/ent/dialect"
	entsql "entgo.io/ent/dialect/sql"
	_ "github.com/jackc/pgx/v5/stdlib"
)

// Open new connection
func Open(databaseUrl string) *ent.Client {
	db, err := sql.Open("pgx", databaseUrl)
	if err != nil {
		log.Fatal(err)
	}

	// Create an ent.Driver from `db`.
	drv := entsql.OpenDB(dialect.Postgres, db)
	return ent.NewClient(ent.Driver(drv))
}

func main() {
	client := Open("postgresql://user:password@127.0.0.1/database")

	// Your code. For example:
	ctx := context.Background()
	if err := client.Schema.Create(ctx); err != nil {
		log.Fatal(err)
	}
	users, err := client.User.Query().All(ctx)
	if err != nil {
		log.Fatal(err)
	}
	log.Println(users)
}
```