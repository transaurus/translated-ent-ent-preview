---
id: grpc-intro
title: gRPC Introduction
sidebar_label: Introduction
---

[gRPC](https://grpc.io) 是由谷歌开源的一款流行RPC框架，基于其内部开发的"Stubby"系统构建。该框架采用[Protocol Buffers](https://developers.google.com/protocol-buffers)——谷歌推出的与语言无关、平台无关的可扩展结构化数据序列化机制。

Ent通过[ent/contrib](https://github.com/ent/contrib)中的插件支持从schema自动生成gRPC服务。

高层次来看，Ent与gRPC的集成工作原理如下：

* 使用名为`entproto`的命令行工具（或代码生成钩子）从ent schema生成protocol buffer定义和gRPC服务定义。通过`entproto`注解对schema进行标注，以辅助实现领域间的映射。
* 通过protobuf编译器插件`protoc-gen-entgrpc`生成gRPC服务定义的实现，该实现使用项目的`ent.Client`进行数据库读写操作。
* 开发者编写嵌入生成服务实现的gRPC服务器。

本教程将使用Ent/gRPC集成构建一个完整可用的gRPC服务器。

### 代码

本教程最终代码可查看[rotemtam/ent-grpc-example](https://github.com/rotemtam/ent-grpc-example)仓库。