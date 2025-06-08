---
title: Visualizing your Data Graph Using entviz
author: Amit Shani
authorURL: "https://github.com/hedwigz"
authorImageURL: "https://avatars.githubusercontent.com/u/8277210?v=4"
authorTwitter: itsamitush
---

加入一个已有大型代码库的项目可能令人望而生畏。

理解应用程序的数据模型是开发者接手现有项目的关键。常用的辅助工具是[ER图（实体关系图）](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model)，它能帮助开发者快速掌握应用的数据结构。

ER图以可视化形式展现数据模型，并详细标注每个实体的字段。Jetbrains DataGrip等工具可通过连接现有数据库自动生成ER图：

<div style={{textAlign: 'center'}}>
  <img alt="Datagrip ER diagram" src="https://entgo.io/images/assets/entviz/datagrip_er_diagram.png" />
  <p style={{fontSize: 12}}>DataGrip ER diagram example</p>
</div>

[Ent](https://entgo.io/docs/getting-started/)是Facebook内部开发的Go语言实体框架，专为处理大型复杂数据模型而设计。通过代码生成提供开箱即用的类型安全和代码补全，既便于理解数据结构又能提升开发效率。如果还能自动生成直观美观的ER图来维护数据模型的宏观视图，岂不更妙？（毕竟谁不喜欢可视化呢？）

### 介绍entviz

[entviz](https://github.com/hedwigz/entviz)是Ent的扩展组件，可自动生成静态HTML页面来可视化数据图谱。

<div style={{textAlign: 'center'}}>
  <img width="600px" alt="Entviz example output" src="https://entgo.io/images/assets/entviz/entviz-example-visualization.png" />
  <p style={{fontSize: 12}}>Entviz example output</p>
</div>

传统ER图工具需连接数据库进行逆向工程，难以保持图表实时性。entviz直接集成Ent schema，无需连接数据库，每次修改schema时都会自动生成最新可视化。

如需了解实现细节，请查看[实现章节](#implementation)。

### 实际演示

首先在entc.go文件中添加entviz扩展：

```bash
go get github.com/hedwigz/entviz
```

:::info
若对`entc`不熟悉，建议阅读[entc文档](https://entgo.io/docs/code-gen#use-entc-as-a-package)了解详情。
:::

```go title="ent/entc.go"
import (
	"log"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"github.com/hedwigz/entviz"
)

func main() {
	err := entc.Generate("./schema", &gen.Config{}, entc.Extensions(entviz.Extension{}))
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

假设我们有个包含用户实体及若干字段的简单schema：

```go title="ent/schema/user.go"
// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
		field.String("email"),
		field.Time("created").
			Default(time.Now),
	}
}
```

现在每次运行以下命令时，entviz都会自动生成图谱可视化：

```bash
go generate ./...
```

此时ent目录下会出现`schema-viz.html`文件：

```bash
$ ll ./ent/schema-viz.html
-rw-r--r-- 1 hedwigz hedwigz 7.3K Aug 27 09:00 schema-viz.html
```

用浏览器打开该HTML文件即可查看可视化效果

![教程图示](https://entgo.io/images/assets/entviz/entviz-tutorial-1.png)

现在我们添加Post实体观察可视化变化：

```bash
ent new Post
```

```go title="ent/schema/post.go"
// Fields of the Post.
func (Post) Fields() []ent.Field {
	return []ent.Field{
		field.String("content"),
		field.Time("created").
			Default(time.Now),
	}
}
```

接着添加从User到Post的[一对多边(O2M)](https://entgo.io/docs/schema-edges/#o2m-two-types)：

```go title="ent/schema/post.go"
// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("posts", Post.Type),
	}
}
```

最后重新生成代码：

```bash
go generate ./...
```

刷新浏览器即可查看更新后的效果！

![教程图示2](https://entgo.io/images/assets/entviz/entviz-tutorial-2.png)

### 实现原理

Entviz 是通过扩展 Ent 的[扩展 API](https://github.com/ent/ent/blob/1304dc3d795b3ea2de7101c7ca745918def668ef/entc/entc.go#L197) 实现的。
Ent 扩展 API 允许您聚合多个[模板](https://entgo.io/docs/templates/)、[钩子](https://entgo.io/docs/hooks/)、[选项](https://entgo.io/docs/code-gen/#code-generation-options)和[注解](https://entgo.io/docs/templates/#annotations)。
例如，entviz 使用模板添加了另一个 Go 文件 `entviz.go`，该文件公开了 `ServeEntviz` 方法，可用作 HTTP 处理器，如下所示：

```go
func main() {
	http.ListenAndServe("localhost:3002", ent.ServeEntviz())
}
```

我们定义了一个扩展结构体，它嵌入了默认扩展，并通过 `Templates` 方法导出我们的模板：

```go
//go:embed entviz.go.tmpl
var tmplfile string
 
type Extension struct {
	entc.DefaultExtension
}
 
func (Extension) Templates() []*gen.Template {
	return []*gen.Template{
		gen.MustParse(gen.NewTemplate("entviz").Parse(tmplfile)),
	}
}
```

模板文件是我们想要生成的代码：

```gotemplate
{{ define "entviz"}}
 
{{ $pkg := base $.Config.Package }}
{{ template "header" $ }}
import (
	_ "embed"
	"net/http"
	"strings"
	"time"
)

//go:embed schema-viz.html
var html string

func ServeEntviz() http.Handler {
	generateTime := time.Now()
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		http.ServeContent(w, req, "schema-viz.html", generateTime, strings.NewReader(html))
	})
}
{{ end }}
```

就这样！现在我们有了一个 ent 包中的新方法。

### 总结

我们了解了 ER 图如何帮助开发者跟踪数据模型。接着，我们介绍了 entviz —— 一个为 Ent 模式自动生成 ER 图的 Ent 扩展。我们看到了 entviz 如何利用 Ent 的扩展 API 来扩展代码生成并添加额外功能。最后，您通过安装并在自己的项目中使用 entviz 看到了它的实际效果。如果您喜欢这段代码或想要贡献 —— 欢迎查看 [GitHub 上的项目](https://github.com/hedwigz/entviz)。

有问题吗？需要入门帮助吗？欢迎加入我们的 [Discord 服务器](https://discord.gg/qZmPgTE6RX) 或 [Slack 频道](https://entgo.io/docs/slack/)。

:::note[获取更多 Ent 新闻和更新：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 在 [Twitter](https://twitter.com/entgo_io) 上关注我们
- 加入 [Gophers Slack](https://entgo.io/docs/slack) 上的 #ent 频道
- 加入 [Ent Discord 服务器](https://discord.gg/qZmPgTE6RX)

:::