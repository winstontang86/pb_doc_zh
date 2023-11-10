# PB最佳实践

经过审查的最佳实践分享。

 客户端和服务器永远不会在完全相同的时间更新，即使你尝试同时更新它们。甚至其中一个可能会回滚。不要假设你可以进行破坏性更改，因为客户端和服务器是同步的。

## 不要重用序号 
永远不要重用序号，这会导致反序列化出错。即使你认为没有人在使用该字段，也不要重用序号。如果更改曾经用过，您的 proto 的序列化版本可能在某个日志中，或者在另一个服务器中可能有旧代码会出错。

## 为已删除的字段保留序号 
当你删除不再使用的字段时，保留其序号，以免将来有人意外地重用它。形如：

reserved 2,3; // 不需要类型。

您还可以保留名称，以避免回收现在已删除的字段名称：

reserved "foo","bar"；

## 为已删除的枚举值保留号码 
当您删除不再使用的枚举值时，保留其号码，以便将来没有人意外地重用它。

reserved 2，3；

您还可以保留名称，以避免回收现在已删除的值名称：

reserved "FOO"，"BAR"；

## 不要更改字段的类型 
永远不要更改字段的类型，这会导致反序列化混乱，就像重用序号一样。protobuf 文档概述了一小部分可以接受的情况（例如，在 int32、uint32、int64 和 bool 之间切换）。即便如此，除非新消息是旧消息的超集，否则更改字段的消息类型将失败。

## 不要添加required字段 
永远不要添加required字段，而是添加 // required 来记录 API 约定。许多人认为required字段是有害的，因此它完全从 proto3 中删除。将所有字段设置为optional或repeated。您永远不知道消息类型会持续多长时间，以及若干年后，当它不再逻辑上需要但 proto 仍然说它需要时，是否有人会被迫用空字符串或零填充您的必需字段。

## 不要创建具有大量字段的message 
不要创建具有“大量”（比如：数百个）字段的消息。在 C++ 中，每个字段都会在内存对象大小中添加大约 65 bits，无论它是否已填充（指针的 8 字节，以及如果字段声明为可选，则位字段中的另一位，用于跟踪字段是否已设置）。当您的 proto 变得过大时，生成的代码甚至可能无法编译（例如，在 Java 中，方法大小有一个硬限制）。

## 在枚举中包含一个unspecified字段 
枚举应在声明中将默认的 FOO_UNSPECIFIED 值作为第一个值。当向 proto2 枚举添加新值时，旧客户端将看到该字段不认识，则当做未设置，getter 将返回默认值或第一个声明的值（如果没有默认值）。为了与 proto 枚举的一致行为，第一个声明的枚举值应该是一个默认的 FOO_UNSPECIFIED 值，并且应该使用标签 0。可能会诱使将此默认值声明为具有语义意义的值，但作为一般规则，不要这样做，以帮助您的协议随着时间的推移添加新的枚举值。同一个pb下声明的所有枚举值都在相同的 C++ 命名空间中，因此在枚举名称前加上前缀字段（比如FOO_）以避免编译错误。如果您永远不需要跨语言常量，int32 将保留未知值并生成较少的代码。**请注意，proto 枚举要求第一个值为零，并且可以在序列化和反序列化的时候跟unknown字段互转。**

【译者建议：枚举有默认值则第一个为默认值（声明中有default字样或注释为default），如果没合适的默认值，那么第一个就不要实际使用，而应当定义为无效值，并用带有 INVALID，UNKNOWN，NONE，UNSPECIFIED 等的词命名，其值设为 0。】

## 不要用 C/C++ 宏常量来作为枚举值（通常是一些保留字） 
使用 C++ 语言已经定义的单词 - 特别是在其头文件（如 math.h）中，如果其中一个头文件的 #include 语句出现在 .proto.h 之前，可能会导致编译错误。避免将宏常量（如 "NULL"、"NAN" 和 "DOMAIN"）用作枚举值。

## 使用已知类型和通用类型 （不要自己造不伦不类的轮子）
强烈建议嵌入以下通用的共享类型。在您的代码中不要使用 int32 timestamp_seconds_since_epoch 或 int64 timeout_millis，当已经存在完全适合的通用类型时！

### 已知类型 
- duration 是有符号的，固定长度的时间跨度（例如 42s）。 
- timestamp 是独立于任何时区或日历的时间点（例如 2017-01-15T01:30:15.01Z）。
- field_mask 是一组符号字段路径（例如 f.b.d）。

### 通用类型 
- Duration 是有符号的，固定长度的时间跨度，如 42s。 
- Timestamp 是独立于任何时区或日历的时间点，如 2017-01-15T01:30:15.01Z。 
- interval 是独立于时区或日历的时间间隔（例如 2017-01-15T01:30:15.01Z - 2017-01-16T02:30:15.01Z）。 
- date 是整个日历日期（例如 2005-09-19）。 
- dayofweek 是一周中的某一天（例如，星期一）。 
- timeofday 是一天中的某个时间（例如 10:42:23）。 
- latlng 是纬度/经度对（例如，37.386051 纬度和 -122.083855 经度）。 
- money 是具有其货币类型的金额（例如 42 美元）。 
- postal_address 是邮政地址（例如 1600 Amphitheatre Parkway Mountain View, CA 94043 USA）。 
- color 是 RGBA 颜色空间中的颜色。 
- month 是一年中的某个月（例如，四月）。

## 把广泛使用的结构体单独用pb文件定义
如果您正在定义希望/担心/期望在您的直接团队之外广泛使用的消息类型或枚举，请考虑将它们放在没有依赖关系的单独文件中。这样，任何人都可以使用这些类型，而无需引入其他 proto 文件中的传递依赖关系。

## 不要更改字段的默认值 
永远不要更改 proto 字段的默认值，这会导致客户端和服务器之间的版本偏差。当它们的构建跨越 proto 更改时，读取未设置值的客户端将看到与读取相同未设置值的服务器不同的结果。基于此Proto3 删除了设置默认值的功能。

## 不要从 Repeated 转为标量 
尽管它不会导致崩溃，但您会丢失数据。对于 JSON，重复性不匹配将丢失整个消息。对于数字 proto3 字段和 proto2 打包字段，从重复转为标量将丢失该字段中的所有数据。对于非数字 proto3 字段和未注释的 proto2 字段，从重复转为标量将导致最后反序列化的值“获胜”。

从标量转为repeated在 proto2 和 proto3 中使用 [packed=false] 是可以的，因为对于二进制序列化，标量值变为一个元素列表。

## 遵循生成代码的样式指南 
在正常代码中引用 Proto 生成的代码。确保 .proto 文件中的选项不会导致生成违反样式指南的代码。例如：

- java_outer_classname 应遵循 https://google.github.io/styleguide/javaguide.html#s5.2.2-class-names
- java_package 和 java_alt_package 应遵循 https://google.github.io/styleguide/javaguide.html#s5.2.1-package-names
- package 虽然在没有 java_package 的情况下用于 Java，但总是直接对应于 C++ 命名空间，因此应遵循 https://google.github.io/styleguide/cppguide.html#Namespace_Names。如果这些样式指南冲突，请使用 Java 的 java_package。

## 不要将文本格式消息用于交换 （用pb序列化的二进制）
基于文本的序列化格式（如文本格式和 JSON）将字段和枚举值表示为字符串。因此，当字段或枚举值被重命名，或者添加新的字段或枚举值或扩展时，使用旧代码反序列化这些格式中的协议缓冲区将失败。尽可能使用二进制序列化进行数据交换，并仅将文本格式用于人类编辑和调试。

如果您在 API 中使用转换为 JSON 的 protos 或用于存储数据，您可能根本无法安全地重命名字段或枚举。

## 永远不要依赖跨构建的序列化稳定性 
proto 序列化的稳定性不能保证在不同的二进制文件或相同二进制文件的不同构建之间。例如，在构建缓存键时，请不要依赖它。

## 不要在与其他代码相同的 Java 包中生成 Java 
Protos 将 Java proto 源代码生成到与手写 Java 源代码分开的包中。package、java_package 和 java_alt_api_package 选项控制生成的 Java 源代码的输出位置。确保手写的 Java 源代码不要也放在同一个包中。常见的做法是将您的 protos 生成到项目中只包含这些 protos（即没有手写源代码）的 proto 子包中。

## 避免使用语言关键字作为字段名 
如果消息、字段、枚举或枚举值的名称是从该字段读取/写入的语言中的关键字，则 protobuf 可能会更改字段名称，并且访问它们的方式可能与普通字段不同。例如，关于 Python 的这个警告。

您还应避免在文件路径中使用关键字，因为这也可能导致问题。

## 附录 
### API 最佳实践 
本文档仅列出了极有可能导致问题的实践。有关如何制定优雅增长的 proto API 的更高级指导，请参阅 [API 最佳实践](https://protobuf.dev/programming-guides/api)。

-----------------------------------------------------
英文原文链接：https://protobuf.dev/programming-guides/dos-donts/

翻译日期：2023-11-10 

原英文内容如下：

# Proto Best Practices

Shares vetted best practices for authoring Protocol Buffers.

Clients and servers are never updated at exactly the same time - even when you try to update them at the same time. One or the other may get rolled back. Don’t assume that you can make a breaking change and it’ll be okay because the client and server are in sync.


## Don’t Re-use a Tag Number
Never re-use a tag number. It messes up deserialization. Even if you think no one is using the field, don’t re-use a tag number. If the change was live ever, there could be serialized versions of your proto in a log somewhere. Or there could be old code in another server that will break.


## Do Reserve Tag Numbers for Deleted Fields
When you delete a field that’s no longer used, reserve its tag number so that no one accidentally re-uses it in the future. Just reserved 2, 3; is enough. No type required (lets you trim dependencies!). You can also reserve names to avoid recycling now-deleted field names: reserved "foo", "bar";.


## Do Reserve Numbers for Deleted Enum Values
When you delete an enum value that’s no longer used, reserve its number so that no one accidentally re-uses it in the future. Just reserved 2, 3; is enough. You can also reserve names to avoid recycling now-deleted value names: reserved "FOO", "BAR";.


## Don’t Change the Type of a Field
Almost never change the type of a field; it’ll mess up deserialization, same as re-using a tag number. The protobuf docs outline a small number of cases that are okay (for example, going between int32, uint32, int64 and bool). However, changing a field’s message type will break unless the new message is a superset of the old one.


## Don’t Add a Required Field
Never add a required field, instead add // required to document the API contract. Required fields are considered harmful by so many they were removed from proto3 completely. Make all fields optional or repeated. You never know how long a message type is going to last and whether someone will be forced to fill in your required field with an empty string or zero in four years when it’s no longer logically required but the proto still says it is.


## Don’t Make a Message with Lots of Fields
Don’t make a message with “lots” (think: hundreds) of fields. In C++ every field adds roughly 65 bits to the in-memory object size whether it’s populated or not (8 bytes for the pointer and, if the field is declared as optional, another bit in a bitfield that keeps track of whether the field is set). When your proto grows too large, the generated code may not even compile (for example, in Java there is a hard limit on the size of a method ).


## Do Include an Unspecified Value in an Enum
Enums should include a default FOO_UNSPECIFIED value as the first value in the declaration . When new values are added to a proto2 enum, old clients will see the field as unset and the getter will return the default value or the first-declared value if no default exists . For consistent behavior with proto enums, the first declared enum value should be a default FOO_UNSPECIFIED value and should use tag 0. It may be tempting to declare this default as a semantically meaningful value but as a general rule, do not, to aid in the evolution of your protocol as new enum values are added over time. All enum values declared under a container message are in the same C++ namespace, so prefix the unspecified value with the enum’s name to avoid compilation errors. If you’ll never need cross-language constants, an int32 will preserve unknown values and generates less code. Note that proto enums require the first value to be zero and can round-trip (deserialize, serialize) an unknown enum value.


## Don’t Use C/C++ Macro Constants for Enum Values
Using words that have already been defined by the C++ language - specifically, in its headers such as math.h, may cause compilation errors if the #include statement for one of those headers appears before the one for .proto.h. Avoid using macro constants such as “NULL,” “NAN,” and “DOMAIN” as enum values.


## Do Use Well-Known Types and Common Types
Embedding the following common, shared types is strongly encouraged. Do not use int32 timestamp_seconds_since_epoch or int64 timeout_millis in your code when a perfectly suitable common type already exists!


### Well-Known Types
duration is a signed, fixed-length span of time (for example, 42s).
timestamp is a point in time independent of any time zone or calendar (for example, 2017-01-15T01:30:15.01Z).
field_mask is a set of symbolic field paths (for example, f.b.d).

### Common Types
Duration is a signed, fixed-length span of time, such as 42s.
Timestamp is a point in time independent of any time zone or calendar, such as 2017-01-15T01:30:15.01Z.
interval is a time interval independent of time zone or calendar (for example, 2017-01-15T01:30:15.01Z - 2017-01-16T02:30:15.01Z).
date is a whole calendar date (for example, 2005-09-19).
dayofweek is a day of week (for example, Monday).
timeofday is a time of day (for example, 10:42:23).
latlng is a latitude/longitude pair (for example, 37.386051 latitude and -122.083855 longitude).
money is an amount of money with its currency type (for example, 42 USD).
postal_address is a postal address (for example, 1600 Amphitheatre Parkway Mountain View, CA 94043 USA).
color is a color in the RGBA color space.
month is a month of year (for example, April).

## Do Define Widely-used Message Types in Separate Files
If you’re defining message types or enums that you hope/fear/expect to be widely used outside your immediate team, consider putting them in their own file with no dependencies. Then it’s easy for anyone to use those types without introducing the transitive dependencies in your other proto files.


## Don’t Change the Default Value of a Field
Almost never change the default value of a proto field. This causes version skew between clients and servers. A client reading an unset value will see a different result than a server reading the same unset value when their builds straddle the proto change. Proto3 removed the ability to set default values.


## Don’t Go from Repeated to Scalar
Although it won’t cause crashes, you’ll lose data. For JSON, a mismatch in repeatedness will lose the whole message. For numeric proto3 fields and proto2 packed fields, going from repeated to scalar will lose all data in that field. For non-numeric proto3 fields and un-annotated proto2 fields, going from repeated to scalar will result in the last deserialized value “winning.”

Going from scalar to repeated is OK in proto2 and in proto3 with [packed=false] because for binary serialization the scalar value becomes a one-element list .


## Do Follow the Style Guide for Generated Code
Proto generated code is referred to in normal code. Ensure that options in .proto file do not result in generation of code which violate the style guide. For example:

- java_outer_classname should follow https://google.github.io/styleguide/javaguide.html#s5.2.2-class-names

- java_package and java_alt_package should follow https://google.github.io/styleguide/javaguide.html#s5.2.1-package-names

- package, although used for Java when java_package is not present, always directly corresponds to C++ namespace and thus should follow https://google.github.io/styleguide/cppguide.html#Namespace_Names. If these style guides conflict, use java_package for Java.


## Don’t use Text Format Messages for Interchange
Text-based serialization formats like text format and JSON represent fields and enum values as strings. As a result, deserialization of protocol buffers in these formats using old code will fail when a field or enum value is renamed, or when a new field or enum value or extension is added. Use binary serialization when possible for data interchange, and use text format for human editing and debugging only.

If you use protos converted to JSON in your API or for storing data, you may not be able to safely rename fields or enums at all.


## Never Rely on Serialization Stability Across Builds
The stability of proto serialization is not guaranteed across binaries or across builds of the same binary. Do not rely on it when, for example, building cache keys.


## Don’t Generate Java Protos in the Same Java Package as Other Code
Generate Java proto sources into a separate package from your hand-written Java sources. The package, java_package and java_alt_api_package options control where the generated Java sources are emitted. Make sure hand-written Java source code does not also live in that same package. A common practice is to generate your protos into a proto subpackage in your project that only contains those protos (that is, no hand-written source code).

## Avoid Using Language Keywords for Field Names
If the name of a message, field, enum, or enum value is a keyword in the language that reads from/writes to that field, then protobuf may change the field name, and may have different ways to access them than normal fields. For example, see this warning about Python.

You should also avoid using keywords in your file paths, as this can also cause problems.

## Appendix
API Best Practices
This document lists only changes that are extremely likely to cause breakage. For higher-level guidance on how to craft proto APIs that grow gracefully see API Best Practices.