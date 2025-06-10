---
id: grpc-intro
title: gRPC Introduction
sidebar_label: Introduction
---

[gRPC](https://grpc.io) 是由谷歌开源的一款流行RPC框架，其前身是谷歌内部开发的"Stubby"系统。它基于[Protocol Buffers](https://developers.google.com/protocol-buffers)——谷歌推出的与语言无关、平台无关的可扩展结构化数据序列化机制。

Ent通过[ent/contrib](https://github.com/ent/contrib)中的插件支持从Schema自动生成gRPC服务。

高层次来看，Ent与gRPC的集成工作流程如下：

* 使用名为`entproto`的命令行工具（或代码生成钩子）从Ent Schema生成Protocol Buffer定义和gRPC服务定义。Schema需通过`entproto`注解来辅助两个领域间的映射。
* 使用protoc（protobuf编译器）插件`protoc-gen-entgrpc`生成gRPC服务实现，该实现会调用项目的`ent.Client`进行数据库读写操作。
* 开发者编写嵌入生成服务实现的gRPC服务器。

本教程将指导您使用Ent/gRPC集成构建一个完整的gRPC服务器。

### 代码

本教程最终代码可查看[rotemtam/ent-grpc-example](https://github.com/rotemtam/ent-grpc-example)仓库。