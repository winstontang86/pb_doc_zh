# Enum Behavior

本文旨在解释 Protocol Buffers 中枚举的当前工作方式与它们应该如何工作。

枚举在不同的语言库中的行为不同。本主题涵盖了不同行为以及将 protobuf 移动到在所有语言中保持一致的想法。如果您想了解有关如何使用枚举的信息，请参阅 proto2 和 proto3 语言指南主题中的相应部分。

## 定义 
枚举有两种不同的风格（开放和封闭）。除了处理unknown字段的方式不同之外，它们的其他行为是相同的。实际上，这意味着简单的情况是相同的，但一些有趣的情况具有有趣的区别（也容易导致bug）。

为了方便阐述，让我们假设我们有以下 .proto 文件（我们现在故意不指定这是 syntax = "proto2" 还是 syntax = "proto3" 文件）：
```protobuf
enum Enum {
  A = 0;
  B = 1;
}

message Msg {
  optional Enum enum = 1;
}
```
开放和封闭之间的区别可以用一个问题来概括：

 当程序解析包含值为 2 的enum字段的二进制数据时会发生什么？

- 开放枚举将解析值 2 并将其直接存储在字段中。访问器将报告字段已设置并返回表示 2 的内容。
- 封闭枚举将解析值 2 并将其存储在消息的未知字段集中。访问器将报告字段未设置并返回枚举的默认值。

## 封闭枚举的影响 

当解析重复字段时，封闭枚举的行为会产生意想不到的后果。当解析重复的 Enum 字段时，所有未知值将放置在未知字段集中。重新序列化时，这些未知值将被再次写入，但不是在列表中的原始位置。

例如，给定 .proto 文件：
```protobuf
enum Enum {
  A = 0;
  B = 1;
}

message Msg {
  repeated Enum r = 1;
}
```
包含值 [0, 2, 1, 2] 的线格式将解析为重复字段包含 [0, 1]，值 [2, 2] 将作为未知字段存储。重新序列化消息后，线格式将对应于 [0, 1, 2, 2]。

类似地，将封闭枚举作为值的映射在值未知时，会将整个条目（键和值）放置在未知字段中。

## 历史 

在引入 syntax = "proto3" 之前，所有枚举都是封闭的（pb2所有枚举都是封闭的）。Proto3 引入了开放枚举，正是因为封闭枚举导致的意外行为。

## 规范 
以下规定了 protobuf 的一致实现的行为。因为这很微妙，许多实现都不符合规范。有关不同实现如何运行的详细信息，请参阅已知问题。

- 当 proto2 文件导入 proto2 文件中定义的枚举时，应将该枚举视为**封闭**。
- 当 proto3 文件导入 proto3 文件中定义的枚举时，应将该枚举视为**开放**。
- 当 proto3 文件导入 proto2 文件中定义的枚举时，protoc 编译器将产生**错误**。
- 当 proto2 文件导入 proto3 文件中定义的枚举时，应将该枚举视为**开放**。

## 已知问题 

### C++ 
所有已知的 C++ 版本都不符合规范。当 proto2 文件导入 proto3 文件中定义的枚举时，C++ 将该字段视为**封闭枚举**。

### C# 
所有已知的 C# 版本都不符合规范。C# 将所有枚举视为开放。

### Java 
所有已知的 Java 版本都不符合规范。当 proto2 文件导入 proto3 文件中定义的枚举时，Java 将该字段视为封闭枚举。

- 注意：Java 处理开放枚举时有一些令人惊讶的边缘情况。给定以下定义：
```protobuf
syntax = "proto3";

enum Enum {
  A = 0;
  B = 1;
}

message Msg {
  repeated Enum name = 1;
}
```
Java 将生成方法 Enum getName() 和 int getNameValue()。对于已知集合之外的值（如 2），getName 方法将返回 Enum.UNRECOGNIZED，而 getNameValue 将返回 2。

类似地，Java 将生成方法 Builder setName(Enum value) 和 Builder setNameValue(int value)。当传递 Enum.UNRECOGNIZED 时，setName 方法将抛出异常，而 setNameValue 将接受 2。

### Kotlin 

所有已知的 Kotlin 版本都不符合规范。当 proto2 文件导入 proto3 文件中定义的枚举时，Kotlin 将该字段视为封闭枚举。

Kotlin 基于 Java 构建，共享其所有奇特之处。

### Go 
所有已知的 Go 版本都不符合规范。Go 将所有枚举视为开放。

。。。。


-----------------------------------------------------
英文原文链接：https://protobuf.dev/programming-guides/enum/

翻译日期：2023-11-9 （winstontang）

原英文内容如下：

## Enum Behavior
Explains how enums currently work in Protocol Buffers vs. how they should work.

Enums behave differently in different language libraries. This topic covers the different behaviors as well as the plans to move protobufs to a state where they are consistent across all languages. If you’re looking for information on how to use enums in general, see the corresponding sections in the proto2 and proto3 language guide topics.

## Definitions
Enums have two distinct flavors (**open and closed**). They behave identically except in their handling of unknown values. Practically, this means that simple cases work the same, but some corner cases have interesting implications.

For the purpose of explanation, let us assume we have the following .proto file (we are deliberately not specifying if this is a syntax = "proto2" or syntax = "proto3" file right now):
```protobuf
enum Enum {
  A = 0;
  B = 1;
}

message Msg {
  optional Enum enum = 1;
}
```
The distinction between open and closed can be encapsulated in a single question:

What happens when a program parses binary data that contains field 1 with the value 2?

**Open enums** will parse the value 2 and store it directly in the field. Accessor will report the field as being set and will return something that represents 2.

**Closed enums** will parse the value 2 and store it in the message’s unknown field set. Accessors will report the field as being unset and will return the enum’s default value.

## Implications of Closed Enums
The behavior of closed enums has unexpected consquences when parsing a repeated field. When a repeated Enum field is parsed all unknown values will be placed in the unknown field set. When it is serialized those unknown values will be written again, but not in their original place in the list. For example, given the .proto file:
```protobuf
enum Enum {
  A = 0;
  B = 1;
}

message Msg {
  repeated Enum r = 1;
}
```
A wire format containing the values [0, 2, 1, 2] for field 1 will parse so that the repeated field contains [0, 1] and the value [2, 2] will end up stored as an unknown field. After reserializing the message, the wire format will correspond to [0, 1, 2, 2].

Similarly, maps with closed enums for their value will place entire entries (key and value) in the unknown fields whenever the value is unknown.

## History
Prior to the introduction of syntax = "proto3" all enums were closed. Proto3 introduced open enums specifically because of the unexpected behavior that closed enums cause.

## Specification
The following specifies the behavior of conformant implementations for protobuf. Because this is subtle, many implementations are out of conformance. See Known Issues for details on how different implementations behave.

When a proto2 file imports an enum defined in a proto2 file, that enum should be treated as closed.
When a proto3 file imports an enum defined in a proto3 file, that enum should be treated as open.
When a proto3 file imports an enum defined in a proto2 file, the protoc compiler will produce an error.
When a proto2 file imports an enum defined in a proto3 file, that enum should be treated as open.

## Known Issues
### C++
All known C++ releases are out of conformance. When a proto2 file imports an enum defined in a proto3 file, C++ treats that field as a closed enum.

### C#
All known C# releases are out of conformance. C# treats all enums as open.

### Java
All known Java releases are out of conformance. When a proto2 file imports an enum defined in a proto3 file, Java treats that field as a closed enum.

NOTE: Java’s handling of open enums has surprising edge cases. Given the following definitions:
```protobuf
syntax = "proto3";

enum Enum {
  A = 0;
  B = 1;
}

message Msg {
  repeated Enum name = 1;
}
```
Java will generate the methods Enum getName() and int getNameValue(). The method getName will return Enum.UNRECOGNIZED for values outside the known set (such as 2), whereas getNameValue will return 2.

Similarly, Java will generate methods Builder setName(Enum value) and Builder setNameValue(int value). The method setName will throw an exception when passed Enum.UNRECOGNIZED, whereas setNameValue will accept 2.

### Kotlin
All known Kotlin releases are out of conformance. When a proto2 file imports an enum defined in a proto3 file, Kotlin treats that field as a closed enum.

Kotlin is built on Java and shares all of its oddities.

### Go
All known Go releases are out of conformance. Go treats all enums as open.

### JSPB
All known JSPB releases are out of conformance. JSPB treats all enums as open.

### PHP
PHP is conformant.

### Python
After 4.22.0, Python is conformant.

In 4.21.x, Python is conformant by default, but setting PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python will cause it to be out of conformance.

Before 4.21.0, Python is out of conformance.

When a proto2 file imports an enum defined in a proto3 file, non-conformant Python versions treat that field as a closed enum.

### Ruby
All known Ruby releases are out of conformance. Ruby treats all enums as open.

### Objective-C
After 22.0, Objective-C is conformant.

Prior to 22.0, Objective-C was out of conformance. When a proto2 file imported an enum defined in a proto3 file, it would treat that field as a closed enum.

### Swift
Swift is conformant.

### Dart
Dart treats all enums as closed.
