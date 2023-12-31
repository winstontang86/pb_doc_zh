# API Best Practices

设计一个未来可扩展的API是出奇的难，本文档在权衡中偏向于长期、无 bug 的演进下提出一下最佳实践建议。 

本文档是上篇[Proto 最佳实践](https://github.com/winstontang86/pb_doc_zh/blob/main/ProtoBestPractices%20PB%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5.md)的补充文章，它不是针对 Java/C++/Go 和其他编程语言的 API 的建议。

如果您在代码review中发现一个 proto 与这些准则有出入，请告知作者这个指导，并帮助传播这个信息。

【译者注：这文章里提的有一半的也不是很好，说得也不太清楚，仅供参考，实际还是看自己业务代码情况】

### 注意 

这些准则仅仅是准则，许多准则都有例外。例如，如果您正在编写性能关键的后端，您可能希望为了速度而牺牲灵活性或安全性。本主题将帮助您更好地了解权衡，并为您的情况做出适当的决策。 

## 精确、简洁地注释大多数字段和消息 

您的 proto 很可能会被那些不了解您在编写或修改时所考虑的人使用用，对您的系统知之甚少的新团队成员或客户有用的术语记录每个字段。

一些具体示例：
```protobuf
// Bad: Option to enable Foo
// Good: Configuration controlling the behavior of the Foo feature.
message FeatureFooConfig {
  // Bad: Sets whether the feature is enabled
  // Good: Required field indicating whether the Foo feature
  // is enabled for account_id.  Must be false if account_id's
  // FOO_OPTIN Gaia bit is not set.
  optional bool enabled;
}

// Bad: Foo object.
// Good: Client-facing representation of a Foo (what/foo) exposed in APIs.
message Foo {
  // Bad: Title of the foo.
  // Good: Indicates the user-supplied title of this Foo, with no
  // normalization or escaping.
  // An example title: "Picture of my cat in a box <3 <3 !!!"
  optional string title [(max_length) = 512];
}

// Bad: Foo config.
// Less-Bad: If the most useful comment is re-stating the name, better to omit
// the comment.
FooConfig foo_config = 3;
```
尽可能简洁地注释每个字段的约束、期望和含义。

您可以使用自定义 proto 注解。请参阅自定义选项以定义跨语言常量，如上例中的 max_length。在 proto2 和 proto3 中都受支持。

随着时间的推移，接口文档可能会变得越来越长。长度会降低清晰度。当文档真的不清楚时，请修复它，但要从整体上看待它，并力求简洁。

## 为接口和存储使用不同的Message（权衡考虑）

如果您向客户端公开的顶级 proto 与您在磁盘上存储的 proto 相同，那么您将面临麻烦。随着时间的推移，越来越多的二进制文件将依赖于您的 API，使其更难改变。您需要有自由更改存储格式的能力，而不影响您的客户端。对代码进行分层，使模块处理客户端 protos、存储 protos 或转换。

为什么？您可能需要更换底层存储系统。您可能希望以不同的方式规范化或反规范化数据。您可能会意识到，您的客户端暴露的 proto 的部分内容适合存储在 RAM 中，而其他部分适合存储在磁盘上。

对于嵌套在顶级请求或响应中的一个或多个级别的 protos，将存储和线上 protos 分开的情况并不那么强烈，这取决于您愿意将客户端与这些 protos 耦合在多大程度上。

维护转换层的成本，但一旦您拥有客户端并需要进行第一次存储更改，它将很快得到回报。

您可能会尝试共享 protos，并在“需要时”发散。由于发散的成本较高，且没有明确的位置放置内部字段，您的 API 将积累客户端不理解或在您不知情的情况下开始依赖的字段。

通过从单独的 proto 文件开始，您的团队将知道在不污染 API 的情况下添加内部字段的位置。在早期阶段，线上 proto 可以与自动转换层（想想：字节复制或 proto 反射）的标签完全相同。Proto 注解也可以为自动转换层提供支持。

**以下是规则的例外场景**：

- 如果 proto 字段是通用类型之一，如 google.type 或 google.protobuf，那么将该类型用作存储和 API 是可以接受的。

- 如果您的服务对性能非常敏感，那么可能值得牺牲灵活性以换取执行速度。如果您的服务没有数百万 QPS 和毫秒级延迟，您可能不是例外。

- 如果以下所有条件都成立：

  - 您的服务是存储系统 
  - 您的系统不会根据客户端的结构化数据做出决策 
  - 您的系统只是在客户端的请求下存储、加载，或者可能提供查询 
请注意，如果您正在实现类似日志系统或通用存储系统的 proto 包装器，那么您可能希望尽可能透明地将客户端的消息传输到存储后端，以免创建依赖关系。考虑使用扩展或通过 Web 安全编码二进制 Proto 序列化将不透明数据编码为字符串。

## 对于变更类接口，不要对一个message完全替换，而是部分更新或仅追加更新 

不要制作一个只接受 完整message Foo 的 UpdateFooRequest接口（意思是不要一下子就一整个message都由client端传来并直接完全替换更新）。

如果客户端不保留未知字段，它们将无法获得 GetFooResponse 的最新字段，从而在往返过程中导致数据丢失。有些系统不保留未知字段。Proto2 和 proto3 的实现会保留未知字段，除非应用程序明确删除未知字段。一般来说，公共 API 应该在服务器端删除未知字段，以防止通过未知字段进行安全攻击。例如，垃圾未知字段可能会导致服务器在将来开始将它们用作新字段时失败。

在没有文档的情况下，处理可选字段是模糊的。UpdateFoo 会清除字段吗？当客户端不知道字段时，这会让您面临数据丢失的风险。它不触及字段吗？那么客户端如何清除字段？两者都不好。

应对方法 #1：使用更新字段掩码 让您的客户端传递它想要修改的字段，并仅在更新请求中包含这些字段。您的服务器将不处理其他字段，仅更新掩码指定的字段。一般来说，您的掩码结构应该反映响应 proto 的结构；也就是说，如果 Foo 包含 Bar，FooMask 包含 BarMask。

应对方法 #2：公开更窄的变更，以改变单个部分 例如，您可以使用 PromoteEmployeeRequest、SetEmployeePayRequest、TransferEmployeeRequest 等，而不是 UpdateEmployeeRequest。

自定义更新方法比非常灵活的更新方法更容易监控、审计和保护，它们也更容易实现和调用。大量的这些方法可能会增加 API 的认知负担（权衡吧）。

## 不要在顶级请求或响应 Proto 中包含原始类型 
本文档中其他部分描述的许多陷阱都可以通过遵循此规则来解决。例如：

通过将重复字段包装在消息中，可以告诉客户端重复字段在存储中未设置还是在此特定调用中未填充。

遵循此规则自然会产生共享请求之间的常见请求选项，读写字段掩码也会受到影响。

您的顶级 proto 几乎总是应该是一个容器，用于容纳其他可以独立增长的消息。

即使您今天只需要一个单一的原始类型，将其包装在消息中也可以为您提供一个明确的扩展路径，并在返回类似值的其他方法之间共享该类型。
例如：
```protobuf
message MultiplicationResponse {
  // Bad: What if you later want to return complex numbers and have an
  // AdditionResponse that returns the same multi-field type?
  optional double result;


  // Good: Other methods can share this type and it can grow as your
  // service adds new features (units, confidence intervals, etc.).
  optional NumericResult result;
}

message NumericResult {
  optional double real_value;
  optional double complex_value;
  optional UnitType units;
}
```
**顶级原始类型的一个例外**：

不透明字符串（或字节）编码一个 proto，但仅在服务器上构建和解析。延续令牌、版本信息令牌和 ID 都可以作为字符串返回，如果字符串实际上是对结构化 proto 的编码。

## 对于当前是二值以后可能会更多情况的场景不要使用布尔值
 
如果您为字段使用布尔值，请确保该字段确实仅描述了两个可能的状态（对于所有时间，而不仅仅是现在和不久的将来）。通常，枚举、整数或消息的灵活性是值得的。

例如，在返回一系列帖子时，开发人员可能需要根据 UX 的当前模型指示是否应根据两列渲染帖子。尽管今天只需要一个布尔值，但没有什么可以阻止 UX 在未来的版本中引入两行帖子、三列帖子或四方帖子。

```protobuf
message GooglePlusPost {
  // Bad: Whether to render this post across two columns.
  optional bool big_post;

  // Good: Rendering hints for clients displaying this post.
  // Clients should use this to decide how prominently to render this
  // post. If absent, assume a default rendering.
  optional LayoutConfig layout_config;
}
message Photo {
  // Bad: True if it's a GIF.
  optional bool gif;

  // Good: File format of the referenced photo (for example, GIF, WebP, PNG).
  optional PhotoType type;
}
```
谨慎地为概念混淆的枚举添加新枚举值。

如果新增字段为枚举引入了新的维度或暗示多个应用程序行为，您几乎肯定需要另起一个字段。

## 少用整数字段作为 ID 

虽然将 int64 用作对象的标识符很有诱惑力，但建议选择使用字符串。

这使您在需要时可以更改 ID 空间，并减少了碰撞的机会，2^64 不再像想象的那么大了。

您还可以将结构化标识符编码为字符串，这会鼓励客户端将其视为不透明的 blob。您仍然必须为字符串提供一个 proto 支持，但您可以将 proto 序列化为字符串字段（编码为Base64），从而从客户端公开的 API 中删除所有内部详细信息。在这种情况下，请遵循以下准则。

```protobuf
message GetFooRequest {
  // Which Foo to fetch.
  optional string foo_id;
}

// Serialized and websafe-base64-encoded into the GetFooRequest.foo_id field.
message InternalFooRef {
  // Only one of these two is set. Foos that have already been
  // migrated use the spanner_foo_id and Foos still living in
  // Caribou Storage Server have a classic_foo_id.
  optional bytes spanner_foo_id;
  optional int64 classic_foo_id;
}
```
如果您从一开始就使用自己的序列化方案来表示字符串中的 ID，事情可能很快变得奇怪。这就是为什么通常最好从支持字符串字段的内部 proto 开始。

## 不要把您期望客户端构造或解析的数据编码到字符串中 

这在传输过程中效率较低，对于 proto 的使用者来说工作量更大，而且对于阅读您文档的人来说会造成困惑。您的客户端还必须考虑编码：列表是用逗号分隔的吗？我是否正确地转义了这个不受信任的数据？数字是十进制的吗？最好让客户端发送实际的消息或原始类型。这在传输过程中更紧凑，对您的客户端更清晰。

当您的服务在多种语言中获得客户端时，这种情况会变得特别糟糕。现在，每个客户端都必须选择正确的解析器或构建器，或者更糟糕的是，编写一个。

更一般地说，选择正确的原始类型。请参阅 Protocol Buffer Language Guide [中的标量值类型表](https://protobuf.dev/programming-guides/proto2/#scalar)。

### 在前端 Proto 中返回 HTML 
对于 JavaScript 客户端，将 HTML 或 JSON 返回到 API 的字段中可能很有诱惑力，这是一个通往将您的 API 与特定 UI 绑定的滑坡。以下是三个具体的危险：

- 一个“敏捷”的非 Web 客户端最终会解析您的 HTML 或 JSON 以获取他们想要的数据，如果您更改格式，这会导致脆弱性，如果他们的解析不佳，将产生漏洞。 
- 如果 HTML 未经过消毒就返回，您的 Web 客户端现在容易受到 XSS 攻击。 
- 您返回的标签和类期望特定的样式表和 DOM 结构。从一个版本到另一个版本，结构会发生变化，您面临版本倾斜问题，其中 JavaScript 客户端比服务器旧，服务器返回的 HTML 在旧客户端上不再正确呈现。对于经常发布的项目来说，这不是一个边缘情况。

除了初始页面加载之外，通常最好返回数据并使用客户端模板在客户端上构建 HTML。

## 将不透明数据用web安全编码方式放到string里面 

如果你在客户端可见的字段中编码了不透明数据（如连续令牌、序列化ID、版本信息等），请在文档中说明客户端应将其视为不透明的数据块。始终使用二进制proto序列化，永远不要使用文本格式或你自己设计的格式来处理这些字段。当你需要扩展在不透明字段中编码的数据时，如果你还没有使用它，你会发现自己在重新发明协议缓冲区序列化。

定义一个内部proto来保存将放入不透明字段的字段（即使你只需要一个字段），将这个内部proto序列化为字节，然后将结果进行web安全的base-64编码，放入你的字符串字段。

使用proto序列化的一个罕见例外：非常偶尔，从精心构造的替代格式中获得的紧凑性优势是值得的。

## 不要包含客户端不会使用的字段 

你向客户端公开的API应仅用于描述如何与你的系统交互，在其中包含其他任何内容都会增加试图理解它的人的认知负担。

在响应proto中返回调试数据曾经是一种常见的做法，但我们现在有了更好的方法。RPC响应扩展（也称为“侧通道”）让你可以用一个proto描述你的客户端接口，用另一个描述你的调试界面。

同样，将实验名称返回在响应proto中曾经是一种方便的记录方式——未明确约定的合同是客户端会在后续操作中将这些实验返回。现在，实现同样的目标的公认方法是在分析管道中进行日志连接。

有一个例外：

如果你需要连续的、实时的分析，并且在小型机器预算上，运行日志连接可能会被禁止。在成本是决定因素的情况下，提前对日志数据进行非规范化处理可能是一种好处。如果你需要将日志数据往返传输，将其作为一个不透明的数据块发送给客户端，并记录请求和响应字段。

**警告**：如果你需要在每个请求上返回或往返隐藏的数据，你就在隐藏使用你的服务的真实成本，这也不好。

## 不要定义没有连续令牌的分页API

```protobuf
message FooQuery {
  // bad：如果数据在第一次查询和第二次查询之间发生变化，这些策略都可能导致你漏掉一些结果数据。在最终一致性的世界中（比如，由Bigtable支持的存储），旧数据在新数据之后出现并不罕见。此外，基于偏移量和页面的方法都假设了一个排序顺序，从而减少了一些灵活性。
  optional int64 max_timestamp_ms;
  optional int32 result_offset;
  optional int32 page_number;
  optional int32 page_size;

  // Good: 你有了灵活性！在FooQueryResponse中返回这个，并让客户端在下一次查询时传回来。
  optional string next_page_token;
}
```
分页API的最佳实践是使用一个由内部proto支持的不透明连续令牌（称为next_page_token），你将其序列化，然后使用WebSafeBase64Escape（C++）或BaseEncoding.base64Url().encode（Java）。那个内部proto可以包含许多字段。重要的是，它为你提供了灵活性，如果你选择，它也可以为你的客户端提供结果的稳定性。

不要忘记验证这个proto的字段作为不可信的输入（参见在字符串中编码不透明数据的注释）。

```protobuf
message InternalPaginationToken {
  // 跟踪到目前为止已经看到的ID。这提供了完美的回忆，但代价是连续令牌更大——尤其是当用户翻页时。
  repeated FooRef seen_ids;

  // 类似于seen_ids策略，但是将seen_ids放在一个Bloom过滤器中，以节省字节并牺牲一些精度。
  optional bytes bloom_filter;

  // 一个合理的第一次尝试，可能会持续更长时间。将它嵌入到一个连续令牌中，让你以后可以在不影响客户端的情况下更改它。
  optional int64 max_timestamp_ms;
}
```
## 将相关字段分组到一个新的消息中，仅嵌套高内聚的字段

```protobuf
message Foo {
  // Bad: The price and currency of this Foo.
  optional int price;
  optional CurrencyType currency;

  // Better: Encapsulates the price and currency of this Foo.
  optional CurrencyAmount price;
}
```
只有高内聚的字段应该被嵌套。如果字段确实相关，你通常会希望在服务器内部一起传递它们。如果它们在一个消息中定义在一起，这会更容易。想想:

CurrencyAmount calculateLocalTax(CurrencyAmount price, Location where)
如果你的CL引入了一个字段，但是那个字段可能以后会有相关的字段，那么预先将它放在自己的消息中，以避免这样的情况:

```protobuf
message Foo {
  // DEPRECATED! Use currency_amount.
  optional int price [deprecated = true];

  // The price and currency of this Foo.
  optional google.type.Money currency_amount;
}
```
嵌套消息的问题在于，虽然CurrencyAmount可能是在API的其他地方重用的热门候选对象，但Foo.CurrencyAmount可能不是。在最糟糕的情况下，Foo.CurrencyAmount被重用，但Foo特定的字段泄漏到其中。

虽然在开发系统时，松散耦合通常被认为是最佳实践，但在设计.proto文件时，这种做法可能并不总是适用。在某些情况下，将两个信息单元紧密耦合（通过将一个单元嵌套在另一个单元内）可能是有意义的。例如，如果您正在创建一组当前看起来相当通用的字段，但您预计稍后会在其中添加专门的字段，那么嵌套消息将阻止其他人从此.proto文件或其他.proto文件中引用该消息。

```protobuf
message Photo {
  // Bad: It's likely PhotoMetadata will be reused outside the scope of Photo,
  // so it's probably a good idea not to nest it and make it easier to access.
  message PhotoMetadata {
    optional int32 width = 1;
    optional int32 height = 2;
  }
  optional PhotoMetadata metadata = 1;
}

message FooConfiguration {
  // Good: Reusing FooConfiguration.Rule outside the scope of FooConfiguration
  // tightly-couples it with likely unrelated components, nesting it dissuades
  // from doing that.
  message Rule {
    optional float multiplier = 1;
  }
  repeated Rule rules = 1;
}
```

## Include a Field Read Mask in Read Requests

```protobuf
// Recommended: use google.protobuf.FieldMask

// Alternative one:
message FooReadMask {
  optional bool return_field1;
  optional bool return_field2;
}

// Alternative two:
message BarReadMask {
  // Tag numbers of the fields in Bar to return.
  repeated int32 fields_to_return;
}
```
If you use

example:
```protobuf
message FooRequestOptions {
  // Field-level read mask of which fields to return. Only fields that
  // were requested will be returned in the response. Clients should only
  // ask for fields they need to help the backend optimize requests.
  optional FooReadMask read_mask;

  // Up to this many comments will be returned on each Foo in the response.
  // Comments that are marked as spam don't count towards the maximum
  // comments. By default, no comments are returned.
  optional int max_comments_to_return;

  // Foos that include embeds that are not on this supported types list will
  // have the embeds down-converted to an embed specified in this list. If no
  // supported types list is specified, no embeds will be returned. If an embed
  // can't be down-converted to one of the supplied supported types, no embed
  // will be returned. Clients are strongly encouraged to always include at
  // least the THING_V2 embed type from EmbedTypes.proto.
  repeated EmbedType embed_supported_types_list;
}

message GetFooRequest {
  // What Foo to read. If the viewer doesn't have access to the Foo or the
  // Foo has been deleted, the response will be empty but will succeed.
  optional string foo_id;

  // Clients are required to include this field. Server returns
  // INVALID_ARGUMENT if FooRequestOptions is left empty.
  optional FooRequestOptions params;
}

message ListFooRequest {
  // Which Foos to return. Searches have 100% recall, but more clauses
  // impact performance.
  optional FooQuery query;

  // Clients are required to include this field. The server returns
  // INVALID_ARGUMENT if FooRequestOptions is left empty.
  optional FooRequestOptions params;
}
```

## Batch/multi-phase Requests
If you start with a repeated message, evolution becomes trivial.
```protobuf
// Describes a type of enhancement applied to a photo
enum EnhancementType {
  ENHANCEMENT_TYPE_UNSPECIFIED;
  RED_EYE_REDUCTION;
  SKIN_SOFTENING;
}

message PhotoEnhancement {
  optional EnhancementType type;
}

message PhotoEnhancementReply {
  // Good: PhotoEnhancement can grow to describe enhancements that require
  // more fields than just an enum.
  repeated PhotoEnhancement enhancements;

  // Bad: If we ever want to return parameters associated with the
  // enhancement, we'd have to introduce a parallel array (terrible) or
  // deprecate this field and introduce a repeated message.
  repeated EnhancementType enhancement_types;
}
```
Imagine
```protobuf
rpc Foo(FooRequest) returns (FooResponse) {
  option deadline = x; // there is no globally good default
}
```
Choosing 

unnatural to pass the two related fields around inside your own service.
```protobuf
message BatchEquationSolverResponse {
  // Bad: Solved values are returned in the order of the equations given in
  // the request.
  repeated double solved_values;
  // (Usually) Bad: Parallel array for solved_values.
  repeated double solved_complex_values;
}

// Good: A separate message that can grow to include more fields and be
// shared among other methods. No order dependence between request and
// response, no order dependence between multiple repeated fields.
message BatchEquationSolverResponse {
  // Deprecated, this will continue to be populated in responses until Q2
  // 2014, after which clients must move to using the solutions field below.
  repeated double solved_values [deprecated = true];

  // Good: Each equation in the request has a unique identifier that's
  // included in the EquationSolution below so that the solutions can be
  // correlated with the equations themselves. Equations are solved in
  // parallel and as the solutions are made they are added to this array.
  repeated EquationSolution solutions;
}
```
Leaking Features Because Your Proto is in a Mobile Build
Android and iOS runtimes both support reflection. To do that, the unfiltered names of fields and messages are embedded in the application binary (APK, IPA) as strings.
```protobuf
message Foo {
  // This will leak existence of Google Teleport project on Android and iOS
  optional FeatureStatus google_teleport_enabled;
}
```

-----------------------------------------------------
英文原文链接：https://protobuf.dev/programming-guides/api/

翻译日期：2023-11-10

原英文内容如下：

# API Best Practices
A future-proof API is surprisingly hard to get right. The suggestions in this document make trade-offs to favor long-term, bug-free evolution.

Updated for proto3. Patches welcome!

This doc is a complement to Proto Best Practices. It’s not a prescription for Java/C++/Go and other APIs.

If you see a proto straying from these guidelines in a code review, point the author to this topic and help spread the word.

### Note
These guidelines are just that and many have documented exceptions. For example, if you’re writing a performance-critical backend, you might want to sacrifice flexibility or safety for speed. This topic will help you better understand the trade-offs and make a decision that works for your situation.

## Precisely, Concisely Document Most Fields and Messages

Chances are good your proto will be inherited and used by people who don’t know what you were thinking when you wrote or modified it. Document each field in terms that will be useful to a new team-member or client with little knowledge of your system.

Some concrete examples:
```protobuf
// Bad: Option to enable Foo
// Good: Configuration controlling the behavior of the Foo feature.
message FeatureFooConfig {
  // Bad: Sets whether the feature is enabled
  // Good: Required field indicating whether the Foo feature
  // is enabled for account_id.  Must be false if account_id's
  // FOO_OPTIN Gaia bit is not set.
  optional bool enabled;
}

// Bad: Foo object.
// Good: Client-facing representation of a Foo (what/foo) exposed in APIs.
message Foo {
  // Bad: Title of the foo.
  // Good: Indicates the user-supplied title of this Foo, with no
  // normalization or escaping.
  // An example title: "Picture of my cat in a box <3 <3 !!!"
  optional string title [(max_length) = 512];
}

// Bad: Foo config.
// Less-Bad: If the most useful comment is re-stating the name, better to omit
// the comment.
FooConfig foo_config = 3;
```
Document the constraints, expectations and interpretation of each field in as few words as possible.

You can use custom proto annotations. See Custom Options to define cross-language constants like max_length in the example above. Supported in proto2 and proto3.

Over time, documentation of an interface can get longer and longer. The length takes away from the clarity. When the documentation is genuinely unclear, fix it, but look at it holistically and aim for brevity.

## Use Different Messages for Wire and Storage

If a top-level proto you expose to clients is the same one you store on disk, you’re headed for trouble. More and more binaries will depend on your API over time, making it harder to change. You’ll want the freedom to change your storage format without impacting your clients. Layer your code so that modules deal either with client protos, storage protos, or translation.

Why? You might want to swap your underlying storage system. You might want to normalize—or denormalize—data differently. You might realize that parts of your client-exposed proto make sense to store in RAM while other parts make sense to go on disk.

When it comes to protos nested one or more levels within a top-level request or response, the case for separating storage and wire protos isn’t as strong, and depends on how closely you’re willing to couple your clients to those protos.

There’s a cost in maintaining the translation layer, but it quickly pays off once you have clients and have to do your first storage changes.

You might be tempted to share protos and diverge “when you need to.” With a perceived high cost to diverge and no clear place to put internal fields, your API will accrue fields clients either don’t understand or begin to depend on without your knowledge.

By starting with separate proto files, your team will know where to add internal fields without polluting your API. In the early days, the wire proto can be tag-for-tag identical with an automatic translation layer (think: byte copying or proto reflection). Proto annotations can also power an automatic translation layer.

The following are exceptions to the rule:

- If the proto field is one of a common type, such as google.type or google.protobuf, then using that type both as storage and API is acceptable.

- If your service is extremely performance-sensitive, it may be worth trading flexibility for execution speed. If your service doesn’t have millions of QPS with millisecond latency, you’re probably not the exception.

- If all of the following are true:

  - your service is the storage system
  - your system doesn’t make decisions based on your clients’ structured data
  - your system simply stores, loads, and perhaps provides queries at your client’s request
Note that if you are implementing something like a logging system or a proto-based wrapper around a generic storages system, then you probably want to aim to have your clients’ messages transit into your storage backend as opaquely as possible so that you don’t create a dependency nexus. Consider using extensions or Encode Opaque Data in Strings by Web-safe Encoding Binary Proto Serialization.

## For Mutations, Support Partial Updates or Append-Only Updates, Not Full Replaces

Don’t make an UpdateFooRequest that only takes a Foo.

If a client doesn’t preserve unknown fields, they will not have the newest fields of GetFooResponse leading to data loss on a round-trip. Some systems don’t preserve unknown fields. Proto2 and proto3 implementations do preserve unknown fields unless the application drops the unknown fields explicitly. In general, public APIs should drop unknown fields on server-side to prevent security attack via unknown fields. For example, garbage unknown fields may cause a server to fail when it starts to use them as new fields in the future.

Absent documentation, handling of optional fields is ambiguous. Will UpdateFoo clear the field? That leaves you open to data loss when the client doesn’t know about the field. Does it not touch a field? Then how can clients clear the field? Neither are good.

### Fix #1: Use an Update Field-mask
Have your client pass which fields it wants to modify and include only those fields in the update request. Your server leaves other fields alone and updates only those specified by the mask. In general, the structure of your mask should mirror the structure of the response proto; that is, if Foo contains Bar, FooMask contains BarMask.

### Fix #2: Expose More Narrow Mutations That Change Individual Pieces
For example, instead of UpdateEmployeeRequest, you might have: PromoteEmployeeRequest, SetEmployeePayRequest, TransferEmployeeRequest, etc.

Custom update methods are easier to monitor, audit, and secure than a very flexible update method. They’re also easier to implement and call. A large number of them can increase the cognitive load of an API.

## Don’t Include Primitive Types in a Top-level Request or Response Proto

Many of the pitfalls described elsewhere in this doc are solved with this rule. For example:

Telling clients that a repeated field is unset in storage versus not-populated in this particular call can be done by wrapping the repeated field in a message.

Common request options that are shared between requests naturally fall out of following this rule. Read and write field masks fall out of this.

Your top-level proto should almost always be a container for other messages that can grow independently.

Even when you only need a single primitive type today, having it wrapped in a message gives you a clear path to expand that type and share the type among other methods that return the similar values. For example:
```protobuf
message MultiplicationResponse {
  // Bad: What if you later want to return complex numbers and have an
  // AdditionResponse that returns the same multi-field type?
  optional double result;


  // Good: Other methods can share this type and it can grow as your
  // service adds new features (units, confidence intervals, etc.).
  optional NumericResult result;
}

message NumericResult {
  optional double real_value;
  optional double complex_value;
  optional UnitType units;
}
```
One exception to top-level primitives: Opaque strings (or bytes) that encode a proto but are only built and parsed on the server. Continuation tokens, version info tokens and IDs can all be returned as strings if the string is actually an encoding of a structured proto.

## Never Use Booleans for Something That Has Two States Now, but Might Have More Later

If you are using boolean for a field, make sure that the field is indeed describing just two possible states (for all time, not just now and the near future). Often, the flexibility of an enum, int, or message turns out to be worth it.

For example, in returning a stream of posts a developer may need to indicate whether a post should be rendered in two-columns or not based on the current mocks from UX. Even though a boolean is all that’s needed today, nothing prevents UX from introducing two-row posts, three-column posts or four-square posts in a future version.
```protobuf
message GooglePlusPost {
  // Bad: Whether to render this post across two columns.
  optional bool big_post;

  // Good: Rendering hints for clients displaying this post.
  // Clients should use this to decide how prominently to render this
  // post. If absent, assume a default rendering.
  optional LayoutConfig layout_config;
}

message Photo {
  // Bad: True if it's a GIF.
  optional bool gif;

  // Good: File format of the referenced photo (for example, GIF, WebP, PNG).
  optional PhotoType type;
}
```
Be cautious about adding states to an enum that conflates concepts.

If a state introduces a new dimension to the enum or implies multiple application behaviors, you almost certainly want another field.

## Rarely Use an Integer Field for an ID

It’s tempting to use an int64 as an identifier for an object. Opt instead for a string.

This lets you change your ID space if you need to and reduces the chance of collisions. 2^64 isn’t as big as it used to be.

You can also encode a structured identifier as a string which encourages clients to treat it as an opaque blob. You still must have a proto backing the string, but you can serialize the proto to a string field (encoded as web-safe Base64) which removes any of the internal details from the client-exposed API. In this case follow the guidelines below.
```protobuf
message GetFooRequest {
  // Which Foo to fetch.
  optional string foo_id;
}

// Serialized and websafe-base64-encoded into the GetFooRequest.foo_id field.
message InternalFooRef {
  // Only one of these two is set. Foos that have already been
  // migrated use the spanner_foo_id and Foos still living in
  // Caribou Storage Server have a classic_foo_id.
  optional bytes spanner_foo_id;
  optional int64 classic_foo_id;
}
```
If you start off with your own serialization scheme to represent your IDs as strings, things can get weird quickly. That’s why it’s often best to start with an internal proto that backs your string field.

## Don’t Encode Data in a String That You Expect a Client to Construct or Parse

It’s less efficient over the wire, more work for the consumer of the proto, and confusing for someone reading your documentation. Your clients also have to wonder about the encoding: Are lists comma-separated? Did I escape this untrusted data correctly? Are numbers base-10? Better to have clients send an actual message or primitive type. It’s more compact over the wire and clearer for your clients.

This gets especially bad when your service acquires clients in several languages. Now each will have to choose the right parser or builder—or worse—write one.

More generally, choose the right primitive type. See the Scalar Value Types table in the Protocol Buffer Language Guide.

### Returning HTML in a Front-End Proto
With a JavaScript client, it’s tempting to return HTML or JSON in a field of your API. This is a slippery slope towards tying your API to a specific UI. Here are three concrete dangers:

- A “scrappy” non-web client will end up parsing your HTML or JSON to get the data they want leading to fragility if you change formats and vulnerabilities if their parsing is bad.
- Your web-client is now vulnerable to an XSS exploit if that HTML is ever returned unsanitized.
- The tags and classes you’re returning expect a particular style-sheet and DOM structure. From release to release, that structure will change, and you risk a version-skew problem where the JavaScript client is older than the server and the HTML the server returns no longer renders properly on old clients. For projects that release often, this is not an edge case.

Other than the initial page load, it’s usually better to return data and use client-side templating to construct HTML on the client .

## Encode Opaque Data in Strings by Web-Safe Encoding Binary Proto Serialization

If you do encode opaque data in a client-visible field (continuation tokens, serialized IDs, version infos, and so on), document that clients should treat it as an opaque blob. Always use binary proto serialization, never text-format or something of your own devising for these fields. When you need to expand the data encoded in an opaque field, you’ll find yourself reinventing protocol buffer serialization if you’re not already using it.

Define an internal proto to hold the fields that will go in the opaque field (even if you only need one field), serialize this internal proto to bytes then web-safe base-64 encode the result into your string field .

One rare exception to using proto serialization: Very occasionally, the compactness wins from a carefully constructed alternative format are worth it.

## Don’t Include Fields that Your Clients Can’t Possibly Have a Use for

The API you expose to clients should only be for describing how to interact with your system. Including anything else in it adds cognitive overhead to someone trying to understand it.

Returning debug data in response protos used to be a common practice, but we have a better way. RPC response extensions (also called “side channels”) let you describe your client interface with one proto and your debugging surface with another.

Similarly, returning experiment names in response protos used to be a logging convenience–the unwritten contract was the client would send those experiments back on subsequent actions. The accepted way of accomplishing the same is to do log joining in the analysis pipeline.

One exception:

If you need continuous, real-time analytics and are on a small machine budget, running log joins might be prohibitive. In cases where cost is a deciding factor, denormalizing log data ahead of time can be a win. If you need log data round-tripped to you, send it to clients as an opaque blob and document the request and response fields.

Caution: If you need to return or round-trip hidden data on every request , you’re hiding the true cost of using your service and that’s not good either.

## Rarely Define a Pagination API Without a Continuation Token

```protobuf
message FooQuery {
  // Bad: If the data changes between the first query and second, each of
  // these strategies can cause you to miss results. In an eventually
  // consistent world (that is, storage backed by Bigtable), it's not uncommon
  // to have old data appear after the new data. Also, the offset- and
  // page-based approaches all assume a sort-order, taking away some
  // flexibility.
  optional int64 max_timestamp_ms;
  optional int32 result_offset;
  optional int32 page_number;
  optional int32 page_size;

  // Good: You've got flexibility! Return this in a FooQueryResponse and
  // have clients pass it back on the next query.
  optional string next_page_token;
}
```
The best practice for a pagination API is to use an opaque continuation token (called next_page_token ) backed by an internal proto that you serialize and then WebSafeBase64Escape (C++) or BaseEncoding.base64Url().encode (Java). That internal proto could include many fields. The important thing is it buys you flexibility and–if you choose–it can buy your clients stability in the results.

Do not forget to validate the fields of this proto as untrustworthy inputs (see note in Encode opaque data in strings).
```protobuf
message InternalPaginationToken {
  // Track which IDs have been seen so far. This gives perfect recall at the
  // expense of a larger continuation token--especially as the user pages
  // back.
  repeated FooRef seen_ids;

  // Similar to the seen_ids strategy, but puts the seen_ids in a Bloom filter
  // to save bytes and sacrifice some precision.
  optional bytes bloom_filter;

  // A reasonable first cut and it may work for longer. Having it embedded in
  // a continuation token lets you change it later without affecting clients.
  optional int64 max_timestamp_ms;
}
```
## Group Related Fields into a New Message. Nest Only Fields with High Cohesion

```protobuf
message Foo {
  // Bad: The price and currency of this Foo.
  optional int price;
  optional CurrencyType currency;

  // Better: Encapsulates the price and currency of this Foo.
  optional CurrencyAmount price;
}
```
Only fields with high cohesion should be nested. If the fields are genuinely related, you’ll often want to pass them around together inside a server. That’s easier if they’re defined together in a message. Think:

CurrencyAmount calculateLocalTax(CurrencyAmount price, Location where)
If your CL introduces one field, but that field might have related fields later, preemptively put it in its own message to avoid this:
```protobuf
message Foo {
  // DEPRECATED! Use currency_amount.
  optional int price [deprecated = true];

  // The price and currency of this Foo.
  optional google.type.Money currency_amount;
}
```
The problem with a nested message is that while CurrencyAmount might be a popular candidate for reuse in other places of your API, Foo.CurrencyAmount might not. In the worst case, Foo.CurrencyAmount is reused, but Foo-specific fields leak into it.

While loose coupling is generally accepted as a best practice when developing systems, that practice may not always apply when designing .proto files. There may be cases in which tightly coupling two units of information (by nesting one unit inside of the other) may make sense. For example, if you are creating a set of fields that appear fairly generic right now but which you anticipate adding specialized fields into at a later time, nesting the message would dissuade others from referencing that message from elsewhere in this or other .proto files.
```protobuf
message Photo {
  // Bad: It's likely PhotoMetadata will be reused outside the scope of Photo,
  // so it's probably a good idea not to nest it and make it easier to access.
  message PhotoMetadata {
    optional int32 width = 1;
    optional int32 height = 2;
  }
  optional PhotoMetadata metadata = 1;
}

message FooConfiguration {
  // Good: Reusing FooConfiguration.Rule outside the scope of FooConfiguration
  // tightly-couples it with likely unrelated components, nesting it dissuades
  // from doing that.
  message Rule {
    optional float multiplier = 1;
  }
  repeated Rule rules = 1;
}
```

## Include a Field Read Mask in Read Requests

```protobuf
// Recommended: use google.protobuf.FieldMask

// Alternative one:
message FooReadMask {
  optional bool return_field1;
  optional bool return_field2;
}

// Alternative two:
message BarReadMask {
  // Tag numbers of the fields in Bar to return.
  repeated int32 fields_to_return;
}
```
If you use the recommended google.protobuf.FieldMask, you can use the FieldMaskUtil (Java/C++) libraries to automatically filter a proto.

Read masks set clear expectations on the client side, give them control of how much data they want back and allow the backend to only fetch data the client needs.

The acceptable alternative is to always populate every field; that is, treat the request as if there were an implicit read mask with all fields set to true. This can get costly as your proto grows.

The worst failure mode is to have an implicit (undeclared) read mask that varies depending on which method populated the message. This anti-pattern leads to apparent data loss on clients that build a local cache from response protos.

## Include a Version Field to Allow for Consistent Reads

When a client does a write followed by a read of the same object, they expect to get back what they wrote–even when the expectation isn’t reasonable for the underlying storage system.

Your server will read the local value and if the local version_info is less than the expected version_info, it will read from remote replicas to find the latest value. Typically version_info is a proto encoded as a string that includes the datacenter the mutation went to and the timestamp at which it was committed.

Even systems backed by consistent storage often want a token to trigger the more expensive read-consistent path rather than incurring the cost on every read.

## Use Consistent Request Options for RPCs that Return the Same Data Type

An example failure pattern is the request options for a service in which each RPC returns the same data type, but has separate request options for specifying things like maximum comments, embeds supported types list, and so on.

The cost of approaching this ad hoc is increased complexity on the client from figuring out how to fill out each request and increased complexity on the server transforming the N request options into a common internal one. A not-small number of real-life bugs are traceable to this example.

Instead, create a single, separate message to hold request options and include that in each of the top-level request messages. Here’s a better-practices example:
```protobuf
message FooRequestOptions {
  // Field-level read mask of which fields to return. Only fields that
  // were requested will be returned in the response. Clients should only
  // ask for fields they need to help the backend optimize requests.
  optional FooReadMask read_mask;

  // Up to this many comments will be returned on each Foo in the response.
  // Comments that are marked as spam don't count towards the maximum
  // comments. By default, no comments are returned.
  optional int max_comments_to_return;

  // Foos that include embeds that are not on this supported types list will
  // have the embeds down-converted to an embed specified in this list. If no
  // supported types list is specified, no embeds will be returned. If an embed
  // can't be down-converted to one of the supplied supported types, no embed
  // will be returned. Clients are strongly encouraged to always include at
  // least the THING_V2 embed type from EmbedTypes.proto.
  repeated EmbedType embed_supported_types_list;
}

message GetFooRequest {
  // What Foo to read. If the viewer doesn't have access to the Foo or the
  // Foo has been deleted, the response will be empty but will succeed.
  optional string foo_id;

  // Clients are required to include this field. Server returns
  // INVALID_ARGUMENT if FooRequestOptions is left empty.
  optional FooRequestOptions params;
}

message ListFooRequest {
  // Which Foos to return. Searches have 100% recall, but more clauses
  // impact performance.
  optional FooQuery query;

  // Clients are required to include this field. The server returns
  // INVALID_ARGUMENT if FooRequestOptions is left empty.
  optional FooRequestOptions params;
}
```

## Batch/multi-phase Requests

Where possible, make mutations atomic. Even more important, make mutations idempotent. A full retry of a partial failure shouldn’t corrupt/duplicate data.

Occasionally, you’ll need a single RPC that encapsulates multiple operations for performance reasons. What to do on a partial failure? If some succeeded and some failed, it’s best to let clients know.

Consider setting the RPC as failed and return details of both the successes and failures in an RPC status proto.

In general, you want clients who are unaware of your handling of partial failures to still behave correctly and clients who are aware to get extra value.

## Create Methods that Return or Manipulate Small Bits of Data and Expect Clients to Compose UIs from Batching Multiple Such Requests

The ability to query many narrowly specified bits of data in a single round-trip allows a wider range of UX options without server changes by letting the client compose what they need.

This is most relevant for front-end and middle-tier servers.

Many services expose their own batching API.

## Make a One-off RPC when the Alternative is Serial Round-trips on Mobile or Web

In cases where a web or mobile client needs to make two queries with a data dependency between them, the current best practice is to create a new RPC that protects the client from the round trip.

In the case of mobile, it’s almost always worth saving your client the cost of an extra round-trip by bundling the two service methods together in one new one. For server-to-server calls, the case may not be as clear; it depends on how performance-sensitive your service is and how much cognitive overhead the new method introduces.

## Make Repeated Fields Messages, Not Scalars or Enums

A common evolution is that a single repeated field needs to become multiple related repeated fields. If you start with a repeated primitive your options are limited–you either create parallel repeated fields, or define a new repeated field with a new message that holds the values and migrate clients to it.

If you start with a repeated message, evolution becomes trivial.
```protobuf
// Describes a type of enhancement applied to a photo
enum EnhancementType {
  ENHANCEMENT_TYPE_UNSPECIFIED;
  RED_EYE_REDUCTION;
  SKIN_SOFTENING;
}

message PhotoEnhancement {
  optional EnhancementType type;
}

message PhotoEnhancementReply {
  // Good: PhotoEnhancement can grow to describe enhancements that require
  // more fields than just an enum.
  repeated PhotoEnhancement enhancements;

  // Bad: If we ever want to return parameters associated with the
  // enhancement, we'd have to introduce a parallel array (terrible) or
  // deprecate this field and introduce a repeated message.
  repeated EnhancementType enhancement_types;
}
```
Imagine the following feature request: “We need to know which enhancements were performed by the user and which enhancements were automatically applied by the system.”

If the enhancement field in PhotoEnhancementReply were a scalar or enum, this would be much harder to support.

This applies equally to maps. It is much easier to add additional fields to a map value if it’s already a message rather than having to migrate from map<string, string> to map<string, MyProto>.

One exception:

Latency-critical applications will find parallel arrays of primitive types are faster to construct and delete than a single array of messages; they can also be smaller over the wire if you use [packed=true] (eliding field tags). Allocating a fixed number of arrays is less work than allocating N messages. Bonus: in Proto3, packing is automatic; you don’t need to explicitly specify it.

## Use Proto Maps

Prior to the introduction in Proto3 of Proto3 maps, services would sometimes expose data as pairs using an ad-hoc KVPair message with scalar fields. Eventually clients would need a deeper structure and would end up devising keys or values that need to be parsed in some way. See Don’t encode data in a string.

So, using a (extensible) message type for the value is an immediate improvement over the naive design.

Maps were back-ported to proto2 in all languages, so using map<scalar, **message**> is better than inventing your own KVPair for the same purpose1.

If you want to represent arbitrary data whose structure you don’t know ahead of time, use google.protobuf.Any.

## Prefer Idempotency

Somewhere in the stack above you, a client may have retry logic. If the retry is a mutation, the user could be in for a surprise. Duplicate comments, build requests, edits, and so on aren’t good for anyone.

A simple way to avoid duplicate writes is to allow clients to specify a client-created request ID that your server dedupes on (for example, hash of content or UUID).

## Be Mindful of Your Service Name, and Make it Globally Unique

A service name (that is, the part after the service keyword in your .proto file) is used in surprisingly many places, not just to generate the service class name. This makes this name more important than one might think.

What’s tricky is that these tools make the implicit assumption that your service name is unique across a network . Worse, the service name they use is the unqualified service name (for example, MyService), not the qualified service name (for example, my_package.MyService).

For this reason, it makes sense to take steps to prevent naming conflicts on your service name, even if it is defined inside a specific package. For example, a service named Watcher is likely to cause problems; something like MyProjectWatcher would be better.

## Ensure Every RPC Specifies and Enforces a (Permissive) Deadline

By default, an RPC does not have a timeout. Since a request may tie up backend resources that are only released on completion, setting a default deadline that allows all well-behaving requests to finish is a good defensive practice. Not enforcing one has in the past caused severe problems for major services . RPC clients should still set a deadline on outgoing RPCs and will typically do so by default when they use standard frameworks. A deadline may and typically will be overwritten by a shorter deadline attached to a request.

Setting the deadline option clearly communicates the RPC deadline to your clients, and is respected and enforced by standard frameworks:
```protobuf
rpc Foo(FooRequest) returns (FooResponse) {
  option deadline = x; // there is no globally good default
}
```
Choosing a deadline value will especially impact how your system acts under load. For existing services, it is critical to evaluate existing client behavior before enforcing new deadlines to avoid breaking clients (consult SRE). In some cases, it may not be possible to enforce a shorter deadline after the fact.

## Bound Request and Response Sizes

Request and response sizes should be bounded. We recommend a bound in the ballpark of 8 MiB, and 2 GiB is a hard limit at which many proto implementations break . Many storage systems have a limit on message sizes .

Also, unbounded messages

- bloat both client and server,
- cause high and unpredictable latency,
- decrease resiliency by relying on a long-lived connection between a single client and a single server.
Here are a few ways to bound all messages in an API:

- Define RPCs that return bounded messages, where each RPC call is logically independent from the others.
- Define RPCs that operate on a single object, instead of on an unbounded, client-specified list of objects.
- Avoid encoding unbounded data in string, byte, or repeated fields.
- Define a long-running operation . Store the result in a storage system designed for scalable, concurrent reads .
- Use a pagination API (see Rarely define a pagination API without a continuation token).
- Use streaming RPCs.
If you are working on a UI, see also Create methods that return or manipulate small bits of data.

## Propagate Status Codes Carefully

RPC services should take care at RPC boundaries to interrogate errors, and return meaningful status errors to their callers.

Let’s examine a toy example to illustrate the point:

Consider a client that calls ProductService.GetProducts, which takes no arguments. As part of GetProducts, ProductService might get all the products, and call LocaleService.LocaliseNutritionFacts for each product.
```protobuf
digraph toy_example {
  node [style=filled]
  client [label="Client"];
  product [label="ProductService"];
  locale [label="LocaleService"];
  client -> product [label="GetProducts"]
  product -> locale [label="LocaliseNutritionFacts"]
}
```
If ProductService is incorrectly implemented, it might send the wrong arguments to LocaleService, resulting in an INVALID_ARGUMENT.

If ProductService carelessly returns errors to its callers, the client will receive INVALID_ARGUMENT, since status codes propagate across RPC boundaries. But, the client didn’t pass any arguments to ProductService.GetProducts. So, the error is worse than useless: it will cause a great deal of confusion!

Instead, ProductService should interrogate errors it receives at the RPC boundary; that is, the ProductService RPC handler it implements. It should return meaningful errors to users: if it received invalid arguments from the caller, it should return INVALID_ARGUMENT. If something downstream received invalid arguments, it should convert the INVALID_ARGUMENT to INTERNAL before returning the error to the caller.

Carelessly propagating status errors leads to confusion, which can be very expensive to debug. Worse, it can lead to an invisible outage where every service forwards a client error without causing any alerts to happen .

The general rule is: at RPC boundaries, take care to interrogate errors, and return meaningful status errors to callers, with appropriate status codes. To convey meaning, each RPC method should document what error codes it returns in which circumstances. The implementation of each method should conform to the documented API contract.

## Appendix
### Returning Repeated Fields
When a repeated field is empty, the client can’t tell if the field just wasn’t populated by the server or if the backing data for the field is genuinely empty. In other words, there’s no hasFoo method for repeated fields.

Wrapping a repeated field in a message is an easy way to get a hasFoo method.

message FooList {
  repeated Foo foos;
}
The more holistic way to solve it is with a field read mask. If the field was requested, an empty list means there’s no data. If the field wasn’t requested the client should ignore the field in the response.

### Updating Repeated Fields
The worst way to update a repeated field is to force the client to supply a replacement list. The dangers with forcing the client to supply the entire array are manyfold. Clients that don’t preserve unknown fields cause data loss. Concurrent writes cause data loss. Even if those problems don’t apply, your clients will need to carefully read your documentation to know how the field is interpreted on the server side. Does an empty field mean the server won’t update it, or that the server will clear it?

Fix #1: Use a repeated update mask that permits the client to replace, delete, or insert elements into the array without supplying the entire array on a write.

Fix #2: Create separate append, replace, delete arrays in the request proto.

Fix #3: Allow only appending or clearing. You can do this by wrapping the repeated field in a message. A present, but empty, message means clear, otherwise, any repeated elements mean append.

### Order Independence in Repeated Fields
Try to avoid order dependence in general. It’s an extra layer of fragility. An especially bad type of order dependence is parallel arrays. Parallel arrays make it more difficult for clients to interpret the results and make it unnatural to pass the two related fields around inside your own service.
```protobuf
message BatchEquationSolverResponse {
  // Bad: Solved values are returned in the order of the equations given in
  // the request.
  repeated double solved_values;
  // (Usually) Bad: Parallel array for solved_values.
  repeated double solved_complex_values;
}

// Good: A separate message that can grow to include more fields and be
// shared among other methods. No order dependence between request and
// response, no order dependence between multiple repeated fields.
message BatchEquationSolverResponse {
  // Deprecated, this will continue to be populated in responses until Q2
  // 2014, after which clients must move to using the solutions field below.
  repeated double solved_values [deprecated = true];

  // Good: Each equation in the request has a unique identifier that's
  // included in the EquationSolution below so that the solutions can be
  // correlated with the equations themselves. Equations are solved in
  // parallel and as the solutions are made they are added to this array.
  repeated EquationSolution solutions;
}
```
### Leaking Features Because Your Proto is in a Mobile Build
Android and iOS runtimes both support reflection. To do that, the unfiltered names of fields and messages are embedded in the application binary (APK, IPA) as strings.
```protobuf
message Foo {
  // This will leak existence of Google Teleport project on Android and iOS
  optional FeatureStatus google_teleport_enabled;
}
```
Several mitigation strategies:

- ProGuard obfuscation on Android. As of Q3 2014. iOS has no obfuscation option: once you have the IPA on a desktop, piping it through strings will reveal field names of included protos. iOS Chrome tear-down
- Curate precisely which fields are sent to mobile clients .
- If plugging the leak isn’t feasible on an acceptable timescale, get buy-in from the feature owner to risk it.
Never use this as an excuse to obfuscate the meaning of a field with a code-name. Either plug the leak or get buy-in to risk it.

### Performance Optimizations
You can trade type safety or clarity for performance wins in some cases. For example, a proto with hundreds of fields–particularly message-type fields–is going to be slower to parse than one with fewer fields. A very deeply-nested message can be slow to deserialize just from the memory management. A handful of techniques teams have used to speed deserialization:

- Create a parallel, trimmed proto that mirrors the larger proto but has only some of the tags declared. Use this for parsing when you don’t need all the fields. Add tests to enforce that tag numbers continue to match as the trimmed proto accumulates numbering “holes.”
- Annotate the fields as “lazily parsed” with [lazy=true].
- Declare a field as bytes and document its type. Clients who care to parse the field can do so manually. The danger with this approach is there’s nothing preventing someone from putting a message of the wrong type in the bytes field. You should never do this with a proto that’s written to any logs, as it prevents the proto from being vetted for PII or scrubbed for policy or privacy reasons.
-----------------------
A gotcha with protos that contain map<k,v> fields. Don’t use them as reduce keys in a MapReduce. The wire format and iteration order of proto3 map items are unspecified which leads to inconsistent map shards. ↩︎