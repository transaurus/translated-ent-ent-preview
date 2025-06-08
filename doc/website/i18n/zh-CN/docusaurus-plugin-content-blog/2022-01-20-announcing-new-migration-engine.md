---
title: "Announcing v0.10: Ent gets a brand-new migration engine"
author: Ariel Mashraki
authorURL: https://github.com/a8m
authorImageURL: https://avatars0.githubusercontent.com/u/7413593
authorTwitter: arielmashraki
---

亲爱的社区成员们，

我非常高兴地宣布Ent新版本v0.10的发布。距离v0.9.1版本已有近六个月时间，因此本次发布包含了大量新功能。在此我想重点介绍我们过去几个月全力打造的重大改进：一个全新的迁移引擎。

### 隆重推出：[Atlas](https://github.com/ariga/atlas)

<img src="https://atlasgo.io/uploads/gopher.svg" width="200" />

Ent现有的迁移引擎表现优异，社区多年来已在生产环境中验证了其诸多巧妙设计。但随着时间推移，原有架构无法解决的问题逐渐累积。同时我们也观察到，现有数据库迁移框架存在诸多不足。过去十年间，基础设施即代码（Infrastructure-as-Code）和声明式配置管理等行业最佳实践的兴起，使得这些早期项目的设计理念显得过时。

鉴于这些问题具有普适性，与具体框架或编程语言无关，我们意识到可以将其构建为通用基础设施解决方案。因此我们不仅重写了Ent迁移引擎，更将其核心能力抽离为独立开源项目——[Atlas](https://atlasgo.io)（[GitHub](https://ariga.io/atlas)）。

Atlas以CLI工具形式分发，采用类似Terraform的HCL新[DDL](https://atlasgo.io/ddl/intro)语法，同时支持作为[Go包](https://pkg.go.dev/ariga.io/atlas)使用。与Ent相同，Atlas采用[Apache 2.0许可证](https://github.com/ariga/atlas/blob/master/LICENSE)。

经过大量开发和测试，Atlas与Ent的集成终于正式可用。这对于提出[#1652](https://github.com/ent/ent/issues/1652)、[#1631](https://github.com/ent/ent/issues/1631)、[#1625](https://github.com/ent/ent/issues/1625)、[#1546](https://github.com/ent/ent/issues/1546)和[#1845](https://github.com/ent/ent/issues/1845)等问题的用户而言是个好消息——这些问题在原有迁移系统中难以妥善解决，现在通过Atlas引擎迎刃而解。

考虑到变更的重大性，当前Atlas迁移引擎采用选择性加入机制。未来我们将逐步过渡到选择性退出模式，并最终弃用现有引擎。这个迁移过程将谨慎推进，根据社区反馈调整节奏。

### 开始使用Ent的Atlas迁移功能

首先升级至最新版Ent：

```shell
go get entgo.io/ent@v0.10.0
```

接着在迁移时使用`WithAtlas(true)`选项启用Atlas引擎：

```go {17}
package main
import (
    "context"
    "log"
    "<project>/ent"
    "<project>/ent/migrate"
    "entgo.io/ent/dialect/sql/schema"
)
func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err = client.Schema.Create(ctx, schema.WithAtlas(true))
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

就是这么简单！

Atlas引擎相较原有实现的重大改进在于其分层架构，清晰分离了***数据库状态检查***(inspection)、***差异分析***(diffing)、***迁移规划***(planning)和***变更执行***(applying)四个阶段。下图展示了Ent集成Atlas的工作流程：

![atlas-migration-process](https://entgo.io/images/assets/migrate-atlas-process.png)

除标准选项（如`WithDropColumn`、`WithGlobalUniqueID`）外，Atlas集成还提供了挂钩到迁移各阶段的扩展能力。

以下两个示例演示如何挂钩到Atlas的`Diff`和`Apply`阶段：

```go
package main
import (
    "context"
    "log"
    "<project>/ent"
    "<project>/ent/migrate"
	"ariga.io/atlas/sql/migrate"
	atlas "ariga.io/atlas/sql/schema"
	"entgo.io/ent/dialect"
	"entgo.io/ent/dialect/sql/schema"
)
func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err := 	client.Schema.Create(
		ctx,
		// Hook into Atlas Diff process.
		schema.WithDiffHook(func(next schema.Differ) schema.Differ {
			return schema.DiffFunc(func(current, desired *atlas.Schema) ([]atlas.Change, error) {
				// Before calculating changes.
				changes, err := next.Diff(current, desired)
				if err != nil {
					return nil, err
				}
				// After diff, you can filter
				// changes or return new ones.
				return changes, nil
			})
		}),
		// Hook into Atlas Apply process.
		schema.WithApplyHook(func(next schema.Applier) schema.Applier {
			return schema.ApplyFunc(func(ctx context.Context, conn dialect.ExecQuerier, plan *migrate.Plan) error {
				// Example to hook into the apply process, or implement
				// a custom applier. For example, write to a file.
				//
				//	for _, c := range plan.Changes {
				//		fmt.Printf("%s: %s", c.Comment, c.Cmd)
				//		if err := conn.Exec(ctx, c.Cmd, c.Args, nil); err != nil {
				//			return err
				//		}
				//	}
				//
				return next.Apply(ctx, conn, plan)
			})
		}),
	)
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

### 下一步计划：v0.11版本

虽然本次版本发布耗时较长，但下一个版本即将到来。以下是v0.11版本的主要规划：

* [支持边/关系模式](https://github.com/ent/ent/issues/1949) - 允许为关系附加元数据字段
* 重构GraphQL集成以完全兼容Relay规范，支持从Ent模式生成GraphQL资源（模式或完整服务端）
* 新增"迁移编写"功能：Atlas库将提供创建"版本化"迁移目录的基础设施（类似Flyway/Liquibase/go-migrate等框架的常见做法）。许多用户已构建了与此类系统的集成方案，我们将基于Atlas为这些流程提供稳健支持
* 查询钩子（拦截器） - 当前钩子仅支持[变更操作](https://entgo.io/docs/hooks/#hooks)，大量用户要求扩展支持读取操作
* 多态边 - 关于多态支持的[议题已开放超一年](https://github.com/ent/ent/issues/1048)。随着Go 1.18泛型特性的落地，我们将重新讨论基于泛型的可能实现方案

### 总结

除了全新迁移引擎的激动公告外，本次发布还包含[42位贡献者提交的199个commit](https://github.com/ent/ent/releases/tag/v0.10.0)，在体量和内容上都具有里程碑意义。Ent作为社区协作的成果，正因为大家的参与而日益精进。在此向所有参与本次版本贡献的成员（按字母排序）致以衷心感谢：

[attackordie](https://github.com/attackordie),
[bbkane](https://github.com/bbkane),
[bodokaiser](https://github.com/bodokaiser),
[cjraa](https://github.com/cjraa),
[dakimura](https://github.com/dakimura),
[dependabot](https://github.com/dependabot),
[EndlessIdea](https://github.com/EndlessIdea),
[ernado](https://github.com/ernado),
[evanlurvey](https://github.com/evanlurvey),
[freb](https://github.com/freb),
[genevieve](https://github.com/genevieve),
[giautm](https://github.com/giautm),
[grevych](https://github.com/grevych),
[hedwigz](https://github.com/hedwigz),
[heliumbrain](https://github.com/heliumbrain),
[hilakashai](https://github.com/hilakashai),
[HurSungYun](https://github.com/HurSungYun),
[idc77](https://github.com/idc77),
[isoppp](https://github.com/isoppp),
[JeremyV2014](https://github.com/JeremyV2014),
[Laconty](https://github.com/Laconty),
[lenuse](https://github.com/lenuse),
[masseelch](https://github.com/masseelch),
[mattn](https://github.com/mattn),
[mookjp](https://github.com/mookjp),
[msal4](https://github.com/msal4),
[naormatania](https://github.com/naormatania),
[odeke-em](https://github.com/odeke-em),
[peanut-cc](https://github.com/peanut-cc),
[posener](https://github.com/posener),
[RiskyFeryansyahP](https://github.com/RiskyFeryansyahP),
[rotemtam](https://github.com/rotemtam),
[s-takehana](https://github.com/s-takehana),
[sadmansakib](https://github.com/sadmansakib),
[sashamelentyev](https://github.com/sashamelentyev),
[seiichi1101](https://github.com/seiichi1101),
[sivchari](https://github.com/sivchari),
[storyicon](https://github.com/storyicon),
[tarrencev](https://github.com/tarrencev),
[ThinkontrolSY](https://github.com/ThinkontrolSY),
[timoha](https://github.com/timoha),
[vecpeng](https://github.com/vecpeng),
[yonidavidson](https://github.com/yonidavidson), 以及
[zeevmoney](https://github.com/zeevmoney)。

此致，
Ariel

:::note[获取更多Ent资讯和更新：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 在[Twitter](https://twitter.com/entgo_io)上关注我们
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 加入[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)

:::