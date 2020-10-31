# 第4章 Encoding and Evolution

每个数据结构的 schema，如果在项目的迭代过程中发生了变化，那么需要留意代码的向前兼容和向后兼容

- 向前兼容：老版本的代码可以读取新的数据结构
- 向后兼容：新版本的代码可以读取旧的数据结构

## 编码

### 为什么程序语言自带的序列化工具不好？

大多程序语言都自带序列化工具，比如 Java 中的 `java.io.Serializable` 、Ruby 中的 `Marshal`、Python 的 `pickle` ，但是他们有下面这些问题

- 序列化的格式通常只有这个程序语言自己能看懂
- 在反序列化的时候，程序会需要实例化任意的类，这样会引来安全性问题
- 难以向前和向后兼容
- CPU 使用率过高，序列化后的文件过大

### JSON，XML，Binary Variants

JSON、XML 和 CSV 的问题

- XML 和 CSV 的语义不能区分字符串和数字；JSON 有数字类型，但是大于 2^53 的数字不能被 IEEE754 标准表示，因此在只有浮点数的语言（比如 JavaScript）中，JSON 不能正确解析这样的超大数字。Twitter 的 API 中会返回 Tweet ID，它会用一个数字和一个字符串来表示它
- JSON 和 XML 可以很好地表示 Unicode 字符串，但是不能表示字节序列，必须使用 Base64 来编码字节序列
- JSON 和 XML 有可选的 schema，CSV 没有 schema

JSON、XML 和 CSV 都是以字符串为基础的编码类型，而使用二进制的编码方式，可以使得编码更快、更紧凑

流行的二进制编码方式有 Apache Thrift 和 Protocol Buffers（protobuf）。他们俩都要求编码时提供一个 schema。下面是 Thrift 的 schema 定义

```c
struct Person {
	1: required string       userName,
	2: optional i64          favoriteNumber,
  3: optional list<string> interests 
}
```

下面是 protobuf 的 schema 定义

```c
message Person {
	required string user_name       = 1;
	optional int64  favorite_number = 2;
  repeated string interests       = 3;
}
```

可以看出来两者十分相似。每个字段对应的数字叫做 field tag

Thrift 其实还有两种编码方式：BinaryProtocol 和 CompactProtocol。后者相对于前者，能编码出更加紧凑的数据。protobuf 的编码方式跟 CompactProtocol 很像

随着项目的开发，schema 在不停地变化。为了实现向后兼容（即让新的代码兼容老的数据），在之后添加到 schema 中的数据字段必须要是 `optional` 的。如果要删除字段，也只能删除 `optional` 的字段

而向前兼容的话，只要不改变每个字段的 tag 就行

如果要改变字段的类型，就必须要查阅相关的文档了。如果从一个数字类型改到另一个数字类型，可能会导致数据丢失精度

Apache Avro 是另一个二进制编码格式，它有两种定义 schema 的方式，一个是 Avro IDL，方便人类能手动编写，另一个是类似于 JSON 的编辑方式，方便于计算机编写

在编码的时候，使用的 schema 叫做 writer's schema；在解码的时候，使用的 schema 叫做 reader's schema。writer's schema 和 reader's schema 并不需要是完全一致的，只需要是相兼容的。于 Thrift 和 Protobuf 基于 field tag 的方式不通，Avro 使用字段的名称。

为了保持向前和向后兼容，在修改 Avro 的 schema 时，只能增加或者删除有默认值的字段。Avro 的默认值是使用一种叫做 `union` 的类型来实现的

因为在解码数据的时候必须有 writer's schema，所以在哪里保存它呢？下面是几个保存的方式

- 如果很多的 record 保存在一个大文件中，这种情况下每个 record 的 writer's schema 都是一样的，那么可以在大文件的开头保存 writer's schema 一次
- 如果是在数据库中，因为数据库的 schema 在不停变化，所以每个 record 的 schema 都不一定一致，这种情况下，每个 record 可以有一个 version 字段，指向写入这个数据时遵从的 writer's schema
- 如果两个进程通过网络来交换数据，那么编码数据的 writer's schema 需要由协议来相互协商

另一个问题是，为什么 Avro 不使用 Thrift 那样的 field tag 来区分字段呢？这是因为 Avro 的方式更适合让计算机来动态创建 schema。比如在数据库中，如果一个表的 schema 发生了变化，那么可以直接从变化后的 schema 创建一个新的 writer's schema。旧的数据读取旧的 writer's schema，新的数据使用新的，因为 Avro 使用 field name 来区分字段。而如果使用 Thrift，则需要手动来给后来新加的字段分配一个 field tag，而且这个 field tag 不能和之前用过的 field tag 重复。比如修改前的 field tag 是 `1,2,3,4` ，现在要删除第 4 个，并增加一个，那么修改后的 field tag 是 `1,2,3,5` 。因此在这个方面，Avro 非常方便

总结一下，像 Thrift、Protocol Buffers 和 Avro 这样的用 schema 定义的二进制编码方式之所以很流行，有下面几个原因

- compact
- self describing，schema 本身就描述了二进制数据。而像 JSON 这样的虽说也可以定义 schema，但是基本没人用，导致 JSON 是 schema-on-read
- schema 能让代码向前和向后兼容
- 利用 schema 能自动生成静态语言的代码

## 数据流的模式

### 数据库

数据库的数据流模式，可以看作是应用程序向未来的自己发送一些消息：现在写，未来读

数据库模式下，数据必须要有向前兼容性，即新版本的应用程序必须能够处理之前老版本的应用程序写入的数据

### REST 与 RPC

REST 是以 HTTP 为基础的 API 设计哲学，它使用了大多数的 HTTP 特性

而另一个种设计风格 SOAP 尽量避免使用 HTTP 的特性，虽然说大多数的 SOAP 请求是建立在 HTTP 之上的。SOAP 使用 XML 来描述请求，相当的复杂

因为 REST 更加简单，因此近年来越发流行

RPC（remote procedure call）想把网络请求包装成本地方法调用。但是因为网络请求与本地方法调用的本质的不同，导致它有下面这些问题

- 本地方法调用会产生确定的结果，而 RPC 因为网络丢包导致非确定的结果
- RPC 会发生超时，客户端无法得知请求是否已经达到服务端
- 网络丢包发生后，如果重复发送一个请求，会导致幂等问题
- 因为网络延迟，RPC 执行时间不确定
- RPC 需要编码参数，如果参数过大，会产生性能开销
- 因为 RPC 的两端可能不是同一种编程语言，这样在数据类型转换方面可能导致精度问题

RPC 继续发展，并不再尝试伪装成本地方法调用，因此出现了 `Future` （ `Promise` ）、 `Stream` 以及服务发现等特性

相比较于 REST，RPC 更适合在企业内部使用，REST 则适合于跨企业的 API 调用

RPC 的前后兼容性取决于编码的数据格式的前后兼容性。如果 RPC 升级后破坏的兼容性，那么可以同时启动多个版本的 RPC 服务端

### 消息队列

消息队列也是一种流行的数据流模式。他的优点有

- 在突发高流量下削峰，此时消息队列会想一个临时缓存一样暂时保存无法处理的消息
- 在消费者崩溃时，重发消息
- 发送端不需要知道消费者的网络地址
- 消息可以发送给多个消费者
- 发送者和消费者结偶，发送者不需要知道有几个消费者、消费者怎么处理、消费者在哪儿等信息

Actor 模式中，一个 Actor 是处理请求的基本单元，不同的 Actor 之间不会共享内存，只通过消息队列交流，因此避免的各种多线程问题

流行的 Actor 框架有 Akka、Orleans 和 Erlang OTP