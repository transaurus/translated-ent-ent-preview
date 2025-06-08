---
title: "What I learned contributing my first feature to Ent's gRPC plugin"
author: Jeremy Vesperman
authorURL: "https://github.com/jeremyv2014"
authorImageURL: "https://avatars.githubusercontent.com/u/9276415?v=4"
image: https://entgo.io/images/assets/grpc/ent_party.png
---

我从事软件开发多年，但直到最近才了解什么是ORM。虽然获得了计算机工程学士学位，但对象关系映射并不在我的学习范围内——当时我过于专注用比特和字节构建底层系统，无暇关注这种高层抽象。因此当我开始参与分布式Web应用开发时，发现自己完全处于知识盲区也就不足为奇了。

为他人开发软件的难点在于，你无法洞悉对方的真实想法。需求往往模糊不清，仅靠提问能获取的理解有限。有时你不得不先构建原型进行演示，才能获得有价值的反馈。

这种方式的弊端显而易见：原型开发耗时且需要频繁调整。如果像我当初那样不了解ORM，就会在重复性工作上浪费大量时间：

1. 根据客户反馈重新定义数据模型
2. 重建测试数据库  
3. 重写数据库交互的SQL语句
4. 重新定义前后端间的gRPC接口
5. 重构前端界面
6. 演示并获取反馈
7. 循环往复

耗费数百小时的工作成果可能面临全面重构，这种挫败感可想而知。当资深开发者问我为何不使用Ent这类ORM时，我的反应既有如释重负，也带着难掩的尴尬。

### 初识Ent

用Ent重构现有数据模型仅用了一天时间。难以置信竟存在如此高效的框架时，我还在手动完成所有工作！通过entproto实现的gRPC集成更是锦上添花——只需在模式中添加注解就能实现基础CRUD操作。这让我直接跳过了从数据模型定义到界面重构的所有中间步骤！但我的使用场景仍存在一个问题：如果事先不知道实体ID，如何通过gRPC接口获取详情？Ent支持查询全部记录，但entproto为何没有提供`GetAll`方法？

### 成为开源贡献者

惊讶地发现该功能确实缺失后，本可以在独立服务中实现该特性，但考虑到这是通用需求，这似乎正是我期待多年的开源贡献机会——既能解决实际问题，又具有普适价值。

在通宵研读entproto源码后，我成功实现了这个功能！怀着成就感提交PR后安然入睡，殊不知已开启了一段深刻的学习之旅。

次日醒来发现PR已被[Rotem](https://github.com/rotemtam)关闭，但获得了共同完善方案的邀请。原因很明确：我的`GetAll`实现存在严重缺陷——仅适用于小规模数据表，在大表上暴露此接口可能导致灾难性后果！

### 可选的接口方法生成

我的解决方案是通过向`entproto.Service()`传参使`GetAll`成为可选功能。我们认同这是个有价值的功能，但应该更具普适性——不能因其最后加入就特殊对待。理想方案是让所有方法都可选生成，例如：

```go
entproto.Service(entproto.Methods(entproto.Create | entproto.Get))
```

然而，为了保持向后兼容性，空的 `entproto.Service()` 注解仍需生成所有方法。作为Go新手，我唯一能想到的实现方式是通过可变参数函数：

```go
func Service(methods ...Method)
```

这种方法的局限性在于只能有一个可变长度的参数类型。如果后续需要为服务注解添加更多选项呢？这时我接触到了强大的[函数式选项设计模式](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)：

```go
// ServiceOption configures the entproto.Service annotation.
type ServiceOption func(svc *service)

// Service annotates an ent.Schema to specify that protobuf service generation is required for it.
func Service(opts ...ServiceOption) schema.Annotation {
	s := service{
		Generate: true,
	}
	for _, apply := range opts {
		apply(&s)
	}
	// Default to generating all methods
	if s.Methods == 0 {
		s.Methods = MethodAll
	}
	return s
}
```

该模式通过接收可变数量的设置函数来配置结构体选项（本例中的服务注解）。基于此模式，我们除了能实现`Methods`外，还可以添加任意数量的其他选项函数。非常巧妙！

### List：更优的GetAll方案

解决可选方法生成问题后，我们重新聚焦于安全实现`GetAll`。Rotem建议参考Google的API改进提案[AIP-132](https://google.aip.dev/132)来实现List方法。这种方案允许客户端获取所有实体，但采用分页机制返回结果。额外好处是——这命名可比"GetAll"优雅多了！

### List请求结构

该设计下的请求消息格式如下：

```protobuf
message ListUserRequest {
  int32 page_size = 1;

  string page_token = 2;

  View view = 3;

  enum View {
    VIEW_UNSPECIFIED = 0;

    BASIC = 1;

    WITH_EDGE_IDS = 2;
  }
}
```

#### 分页大小

`page_size`字段允许客户端指定单页最大返回条目数（上限1000），既解决了原始`GetAll`可能返回过量数据的问题，又通过限制最大页数避免服务器过载。

#### 分页令牌

`page_token`是base64编码字符串，服务器用它确定下一页起始位置。空令牌表示获取第一页。

#### 视图模式

`view`字段用于指定是否在响应中返回实体关联的边缘ID。

### List响应结构

响应消息格式如下：

```protobuf
message ListUserResponse {
  repeated User user_list = 1;

  string next_page_token = 2;
}
```

#### 实体列表

`user_list`字段包含当前页的实体集合。

#### 下一页令牌

`next_page_token`是base64编码字符串，用于获取下一页数据。空令牌表示当前页为最后一页。

### 分页实现

确定gRPC接口后，面临分页实现的技术选型。最直观的方案是使用`LIMIT/OFFSET`分页跳过已读条目，但这种方式存在严重[缺陷](https://use-the-index-luke.com/no-offset)——数据库必须读取所有跳过的行才能定位目标数据。

#### 键集分页

Rotem提出了更优方案：键集分页。虽然需要借助唯一列（或列组合）进行排序稍显复杂，但能带来显著的性能提升。该方案利用已排序的特性，只需筛选唯一列值大于（升序）或小于（降序）客户端令牌指定值的条目。这意味着数据库无需读取跳过的行，极大提升了大数据表查询效率！

选定键集分页方案后，下一步需要确定实体排序方式。对Ent框架而言最直接的方式是使用`id`字段——所有模式都包含该字段且保证唯一性。我们选择将此作为初始实现方案。此外还需决定采用升序或降序排列，初始版本选择了降序排序。

### 使用方式

下面演示新`List`功能的具体用法：

```go
package main

import (
  "context"
  "log"

  "ent-grpc-example/ent/proto/entpb"
  "google.golang.org/grpc"
  "google.golang.org/grpc/status"
)

func main() {
  // Open a connection to the server.
  conn, err := grpc.Dial(":5000", grpc.WithInsecure())
  if err != nil {
    log.Fatalf("failed connecting to server: %s", err)
  }
  defer conn.Close()
  // Create a User service Client on the connection.
  client := entpb.NewUserServiceClient(conn)
  ctx := context.Background()
  // Initialize token for first page.
  pageToken := ""
  // Retrieve all pages of users.
  for {
    // Ask the server for the next page of users, limiting entries to 100.
    users, err := client.List(ctx, &entpb.ListUserRequest{
        PageSize:  100,
        PageToken: pageToken,
    })
    if err != nil {
        se, _ := status.FromError(err)
        log.Fatalf("failed retrieving user list: status=%s message=%s", se.Code(), se.Message())
    }
    // Check if we've reached the last page of users.
    if users.NextPageToken == "" {
        break
    }
    // Update token for next request.
    pageToken = users.NextPageToken
    log.Printf("users retrieved: %v", users)
  }
}
```

### 未来展望

当前`List`实现存在若干待改进的限制：首先排序仅支持`id`列，虽保证模式兼容性但缺乏灵活性。理想情况下应允许客户端指定排序列，或在模式中定义排序列。其次当前仅支持降序排列，未来可将其设为请求选项。最后目前仅支持`int32`、`uuid`和`string`类型的`id`字段，因为代码生成模板中需要为每种类型单独定义页面令牌的转换方法（毕竟个人开发精力有限）。

### 总结

初次贡献entproto功能时我颇为忐忑，作为开源新手不知将面临什么。但很高兴地告诉大家，参与Ent项目充满乐趣！我有幸与优秀的开发者共事，同时为开源社区贡献力量。从函数式选项、键集分页到代码审查中的细节洞察，整个过程让我对Go语言（及软件开发整体）有了更深理解。强烈建议有意贡献者勇敢尝试——你将从实践中收获超乎想象的成长！

若有疑问或需要入门帮助，欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack/)。

:::note[获取更多Ent资讯：]

- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::