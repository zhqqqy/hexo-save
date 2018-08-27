---
title: protobuf
date: 2018-08-27 13:48:58
tags: protobuf，google
categories: protobuf
---

# 简介

由于线上项目在做压力测试的时候，发现在数据量不是特别大的时候cpu资源消耗特别多，经过go pprof工具的cpu性能分析发现，因为在此项目中有大量使用gob序列化和发序列化的地方导致，因为gob序列化需要用到反射，性能很差，最后在做了各个序列化对比后，发现google的Protocol Buffers序列化与反序列化性能消耗差不多，性能是gob的十几倍。因此想将线上数据改造为protobuf格式。

Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。

## 安装 Google Protocol Buffer

- protoc可执行文件下载地址：<https://github.com/google/protobuf/releases>

- 安装编译器插件protoc-gen-go (protoc-gen-go用于生成Go语言代码)

  ```shell
  go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
  ```

## 编写.proto文件

首先我们需要编写一个 proto 文件，定义我们程序中需要处理的结构化数据，在 protobuf 的术语中，结构化数据被称为 Message。proto 文件非常类似 java 或者 C 语言的数据定义。代码清单 1 显示了例子应用中的 proto 文件内容。

##### 清单 1. proto 文件

```protobuf
syntax = "proto3";
package lm; 
message helloworld 
{ 
   required int32     id = 1;  // ID 
   required string    str = 2;  // str 
   optional int32     opt = 3;  //optional field 
}
```

一个比较好的习惯是认真对待 proto 文件的文件名。比如将命名规则定于如下：

`packageName.MessageName.proto`

在上例中，package 名字叫做 lm，定义了一个消息 helloworld，该消息有三个成员，类型为 int32 的 id，另一个为类型为 string 的成员 str。opt 是一个可选的成员，即消息中可以不包含该成员。

### 编译 .proto 文件

写好 proto 文件之后就可以用 Protobuf 编译器将该文件编译成目标语言了。本例中我们将使用 Go。

**这里详细介绍golang的编译姿势:**

- `-I` 参数：指定import路径，可以指定多个`-I`参数，编译时按顺序查找，不指定时默认查找当前目录
- `--go_out` ：golang编译支持，支持以下参数
  - `plugins=plugin1+plugin2` - 指定插件，目前只支持grpc，即：`plugins=grpc`
  - `M` 参数 - 指定导入的.proto文件路径编译后对应的golang包名(不指定本参数默认就是`.proto`文件中`import`语句的路径)
  - `import_prefix=xxx` - 为所有`import`路径添加前缀，主要用于编译子目录内的多个proto文件，这个参数按理说很有用，尤其适用替代一些情况时的`M`参数，但是实际使用时有个蛋疼的问题导致并不能达到我们预想的效果，自己尝试看看吧
  - `import_path=foo/bar` - 用于指定未声明`package`或`go_package`的文件的包名，最右面的斜线前的字符会被忽略
  - 末尾 `:编译文件路径 .proto文件路径(支持通配符)`

完整示例：

假设您的 proto 文件存放在 $SRC_DIR 下面，您也想把生成的文件放在同一个目录下，则可以使用如下命令：

`protoc -I=$SRC_DIR --go_out=$DST_DIR $SRC_DIR/addressbook.proto`

命令将生成一个文件：

`addressbook.pb.go` 



















