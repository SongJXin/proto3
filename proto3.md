原文地址：[Protocol Buffers proto3](https://developers.google.com/protocol-buffers/docs/proto3)  
[toc]  
本指南讲述如何使用 protocol buffer 语言来构造 portocol buffer数据。包括 `.proto` 文件语法和如何从`.proto`文件生成数据访问类。它涵盖了`protocol buffers`语言的**proto3**版本:有关旧的**proto2**语法的信息，请参阅[proto2语言指南](https://developers.google.com/protocol-buffers/docs/proto)。  
# 定义消息类型  
首先让我们看一个非常简单的例子。假设您想定义一个搜索请求消息格式，其中每个搜索请求都有一个查询字符串、您感兴趣的结果的特定页面以及每个页面的一些结果。下面是用于定义消息类型的`.proto`文件。
```
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```
* 文件的第一行指定您使用的是`proto3`语法:如果不这样做，`protocol buffers`编译器将假定您使用的是[proto2](https://developers.google.com/protocol-buffers/docs/proto)。这必须是文件的第一个非空、非注释行。  
* `SearchRequest`信息定义了三个字段(名称-值 对)，每个你想要的数据都包含在这种类型的信息中。每个字段都有一个名称和类型。  
## 指定字段类型  
在上面的例子中，所有字段都是[标量类型](#sca):两个整数(`page_number`和`result_per_page`)和一个字符串(`query`)。但是，您还可以为字段指定复合类型，包括[枚举](#enu)和其他消息类型。
## 分配字段号
如您所见，消息定义中的每个字段都有一个**唯一的编号**。这些字段编号作为[消息二进制格式化](https://developers.google.com/protocol-buffers/docs/encoding)后的标识字段，一旦消息类型被使用,这些数字就不要修改。注意，范围为1到15的字段号需要一个字节进行编码，包括字段号和字段类型(您可以在[Protocol Buffers编码](https://developers.google.com/protocol-buffers/docs/encoding.html#structure)中了解更多)。16到2047范围内的字段号占用两个字节。因此，您应该为频繁出现的消息元素保留数字1到15。记得为将来可能添加的频繁出现的元素留出一些空间。可以指定的最小字段号是1，最大字段号是2的29次方减1，也就是536870911。但是您不能使用19000 到 19999(`FieldDescriptor::kFirstReservedNumber` 到 `FieldDescriptor::kLastReservedNumber`),因为它们是预留给Protocol Buffers 实现的。如果您在`.proto`中使用了它们，Protocol Buffer编译器会报错。类似地，您不能使用任何先前[保留的字段号](#reserved-field)。
## 指定字段规则
消息字段可以是下列字段之一:
* `singular`:格式良好的消息可以有零个或一个(但不能超过一个)。
* `repeated`:此字段可以在格式良好的消息中重复任意次数(包括零次)。重复值的顺序将被保留。
在proto3中，标量类型的`repeated`字段默认使用`packed`编码。  
你可以从[ Protocol Buffer 编码](https://developers.google.com/protocol-buffers/docs/encoding.html#packed)中找到更多关于`packed`编码的信息。
## 添加更多信息类型
可以在一个`.proto`文件中定义多个消息类型。这对于定义多个相关消息非常有用——例如，如果您想定义与`SearchResponse`消息类型相对应的回复消息格式，可以将其添加到相同的`.proto`:  
```
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message SearchResponse {
 ...
}
```

## 添加注释
要向`.proto`文件添加注释，请使用C/ c++风格的//和/*…* /语法。  
```
/* SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response. */

message SearchRequest {
  string query = 1;
  int32 page_number = 2;  // Which page number do we want?
  int32 result_per_page = 3;  // Number of results to return per page.
}
```
## 预留字段<a id=reserved-field></a>
如果您通过完全删除或注释某个字段来[更新](#update)消息类型，未来的用户可以在对该类型进行更新时重用字段号。这会导致如果有人使用了旧的`.proto`版本，会造成包括数据损坏、隐私漏洞等问题。避免这个问题的一个方法就是将这些字段号(和/或名字，这也会导致json序列化产生问题)设置为`reserved`。如果将来有任何用户试图使用这些字段标识符，protocol buffer编译器将会发出警告。
```
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```
注意：不能在同一条`reserved`字段中混合使用字段名和字段号。
## `.proto`文件生成了什么
当你用 protocol buffer 编译器编译`.proto`文件，编译器用您选择的语言和您需要使用您在文件中描述的消息类型生成代码，包括获取和设置字段值，将消息序列化到输出流，以及从输入流解析消息。
* 对于**c++**，编译器从每个`.proto`生成一个`.h`和`.cc`文件，并为文件中描述的每种消息类型提供一个类。
* 对于**Java**，编译器生成一个`.Java`文件，其中包含每个消息类型的类，以及用于创建消息类实例的特殊构建器类。
* **Python**略有不同——Python编译器使用`.proto`中的每种消息类型的静态描述符生成模块，然后将其与元类一起使用，在运行时创建必要的Python数据访问类。
* 对于**Go**，编译器生成`.pb.go`。文件中每个消息类型都有一个类型。
* 对于**Ruby**，编译器使用包含消息类型的Ruby模块生成`.rb`文件。
* 对于**Objective-C**，编译器为每个`.proto`生成一个`pbobjc.h`和`pbobjc.m`。并为文件中描述的每种消息类型提供一个类。
* 对于**c#**，编译器从每个`.proto`生成一个`.cs`文件，并为文件中描述的每种消息类型提供一个类。
* 对于**Dart**，编译器生成一个`.pb.dart`。文件中的每个消息类型都有一个对应的类。
有关更多API细节，请参见[相关API参考](https://developers.google.com/protocol-buffers/docs/reference/overview)
# 标量(Scalar)值类型<a id=sca></a>
标量消息字段可以有以下类型之一,下表显示了`.proto`文件中指定的类型，以及自动生成的类中相应的类型:  

|.proto Type|Notes|C++ Type|JAVA Type|Python Type[2]|Go Type|Ruby Type|C# Type|PHP Type|Dart Type|
|------|------|------|------|------|------|------|------|------|------|
|double| |double|double|float|float64|Float|double|float|double|
|float| |float|float|float|float32|Float|float|float|double|
|int32|变长编码。负数编码效率低,有负数请使用sint32|int32|int|int|int32|Fixnum / Bignum |int|integer|int|
|int64|变长编码。负数编码效率低,有负数请使用sint64|int64|long|int/long[3]|int64|Bignum |long|integer/string[5]|int64|
|uint32|变长编码|uint32|int[1]|int/long[3]|uint32|Fixnum or Bignum|uint|integer|int|
|uint64|变长编码|uint64|long[1]|int/long[3]|uint64|Bignum|ulong|integer/string[5]|Int64|
|sint32|变长编码|int32|int|int|int32|Fixnum / Bignum |int|integer|int|
|sint64|变长编码|int64|long|int/long[3]|int64|Bignum |long|integer/string[5]|int64|
|fixed32|总是4个字节，如果值总是大于2的28次方，比uint32更有效|uint32|int[1]|int/long[3]|uint32|Fixnum or Bignum|uint|integer|int|
|fixed64|总是8个字节，如果值总是大于2的56次方，比uint64更有效|uint64|long[1]|int/long[3]|uint64|Bignum|ulong|integer/string[5]|Int64|
|sfixed32|总是4个字节|int32|int|int|int32|Fixnum or Bignum|int|integer|int|
|sfixed64|总是8个字节|int64|long|int/long[3]|int64|Bignum |long|integer/string[5]|int64|
|bool| |bool|boolean|bool|bool|TrueClass/FalseClass|bool|boolean|bool|
|string|必须始终包含UTF-8编码或7位ASCII文本|sting|String|str/unicode[4]|string|String (UTF-8)|string|string|String|
|bytes|可以包含任意字节序列|string|ByteString|str|[]byte|String (ASCII-8BIT)|ByteString|string|List<int>|
  
你可以在[ Protocol Buffer 编码](https://developers.google.com/protocol-buffers/docs/encoding)中了解更多当你序列化信息的时候，它们是如何被编码的。  
* [1]在java中，没有无符号整型
* [2]在所有情况下，为字段设置值将执行类型检查，以确保它是有效的。
* [3]64位或无符号的32位整数在解码时总是表示为长，但如果在设置字段时给定int，则可以表示为int。在所有情况下，值必须适合设置时表示的类型。见[2]
* [4]Python字符串在decode上表示为unicode，但如果给定ASCII字符串，则可以是str(这可能会发生变化)。
* [5]Integer用于64位机器，string用于32位机器。
# 默认值<a id=default></a>
当解析消息时，如果编码的消息不包含特定的奇异元素，则解析对象中的对应字段将设置为该字段的默认值。These defaults are type-specific:
* 字符串的默认值是空字符串。
* 字节型的默认值是空字节。
* 布尔类型的默认值是false。
* 数值类型默认值是0。
* 枚举类型默认值是第一个定义的枚举值，它必须为0。
* 对于消息字段，未设置该字段。其确切值依赖于语言。有关详细信息，请参阅[生成代码指南](https://developers.google.com/protocol-buffers/docs/reference/overview)。
重复字段的默认值为空(通常是适当语言中的空列表)。  
注意,对于标量消息字段,一旦消息被解析，它将无法告诉你一个字段是否显式地设置为默认值(例如一个布尔值是否设置为false)或不设置:在定义你的消息类型时你应该牢记这一点。例如，如果你不想某个事件在默认情况下也发生，就不要设置事件的触发条件时bool值为false的时候。还要注意，如果将标量消息字段设置为其默认值，则该值不会在线上序列化。  
有关默认值如何在生成的代码中工作的更多细节，请参阅所选语言的[生成代码指南](https://developers.google.com/protocol-buffers/docs/reference/overview)。

# 枚举类型(Enumerations)<a id=enu></a>
在定义消息类型时，您可能希望它的一个字段只有一个预定义的值列表。例如，如果你想为`SearchRequest`添加一个`corpus`字段，`·这个corpus`可以是 `UNIVERSAL`, `WEB`, `IMAGES`, `LOCAL`, `NEWS`, `PRODUCTS` 或者 `VIDEO`。您可以非常简单地将`enum`添加到消息定义中，并为每个可能的值添加一个常量。  
在下面的例子中，我们添加了一个名为`Corpus`的`enum`，其中包含所有可能的值，以及一个`Corpus`类型的字段:
```
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 4;
}
```
如您所见，`Corpus`枚举的第一个常量映射到零:每个枚举定义必须包含一个映射到零的常量作为其第一个元素。这是因为:  
* 必须有一个0值，这样我们才能使用0作为一个数值[默认值](#default)。
* 0值需要是第一个元素，以便与[proto2](https://developers.google.com/protocol-buffers/docs/proto)语义兼容，proto2第一个enum值总是默认值。  
您可以通过将相同的值赋给不同的枚举常量来定义别名。为此，您需要将`allow_alias`选项设置为`true`，否则当protocol编译器找到别名时，会报错。  
```
enum EnumAllowingAlias {
  option allow_alias = true;
  UNKNOWN = 0;
  STARTED = 1;
  RUNNING = 1;
}
enum EnumNotAllowingAlias {
  UNKNOWN = 0;
  STARTED = 1;
  // RUNNING = 1;  // Uncommenting this line will cause a compile error inside Google and a warning message outside.
}
```
枚举数常量必须在32位整数的范围内。由于枚举值在线路上使用可变值编码，负值效率低下，因此不推荐使用。您可以在消息定义中定义枚举，如上面的示例所示，也可以在消息定义之外定义枚举—这些枚举可以在`.proto`文件中的任何消息定义中重用。您还可以使用`MessageType.EnumType`语法，在一条消息中声明的枚举类型作为另一条消息中的字段类型。  
当你在`.proto`文件上运行`protocol buffer`编译器的时候，生成的代码将为Java或c++提供相应的枚举，为python生成一个特殊的`EnumDescriptor `类，它被用于在运行时生成的类中创建一组具有整数值的符号常量。  
尽管在反序列化消息时如何表示这些值取决于语言，但是在反序列化期间，未识别的枚举值将保存在消息中。在支持具有超出指定符号范围的值的开枚举类型的语言中，例如c++和Go，未知的枚举值只是作为其底层整数表示形式存储。具有闭枚举类型(如Java)的n种语言中，枚举中的一个大小写用于表示未识别的值，并且可以使用特殊的访问器访问底层整数。在这两种情况下，如果消息被序列化，未识别的值仍将与消息一起序列化。  
有关如何在应用程序中使用消息枚举的更多信息，请参阅所选语言的[生成代码指南](https://developers.google.com/protocol-buffers/docs/reference/overview)。
## 预留值
如果您通过完全删除枚举条目或注释它来[更新](#update)枚举类型，未来的用户可以在对该类型进行更新时重用该数值。这会导致如果有人使用了旧的`.proto`版本，会造成包括数据损坏、隐私漏洞等问题。避免这个问题的一个方法就是将这些字段号(和/或名字，这也会导致json序列化产生问题)设置为`reserved`。如果将来有任何用户试图使用这些字段标识符，protocol buffer编译器将会发出警告。您可以使用`max`关键字指定您的保留数值范围上升到可能的最大值。
```
enum Foo {
  reserved 2, 15, 9 to 11, 40 to max;
  reserved "FOO", "BAR";
}
```
注意：不能在同一条`reserved`字段中混合使用字段名和字段号。
# 使用其他消息类型<a id=other></a>
您可以使用其他消息类型作为字段类型。例如，假设您希望在每个`SearchResponse`消息中包含`Result`消息——为此，您可以在相同的`.proto`中定义一个结果消息类型，然后在`SearchResponse`中指定一个`Result`类型字段
```
message SearchResponse {
  repeated Result results = 1;
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```
## 导入定义
在上述例子中，结果消息类型在与`SearchResponse`相同的文件中定义,如果您想要用作字段类型的消息类型已经在另一个`.proto`文件中定义了呢？  
通过导入其他`.proto`文件，可以使用它们的定义。要导入另一个`.proto`定义，需要在文件顶部添加`import`语句.
```
import "myproject/other_protos.proto";
```
默认情况下，您只能使用直接导入的`.proto`文件中的定义。但是，有时可能需要将.proto文件移动到新的位置。与直接移动.proto文件并在一次更改中更新所有调用站点不同，现在可以在旧位置中放置一个虚构的.proto文件，使用import public概念将所有导入转发到新位置。导入包含`improt public`语句的proto的任何人都可以临时依赖`import public`依赖项。例如：
```
// new.proto
// All definitions are moved here
```
```
// old.proto
// This is the proto that all clients are importing.
import public "new.proto";
import "other.proto";
```
```
// client.proto
import "old.proto";
// You use definitions from old.proto and new.proto, but not other.proto
```
`protocol`编译器使用`-I/——proto_path`标志在`protocol`编译器命令行指定的一组目录中搜索导入的文件,如果没有给出标志，它将在调用编译器的目录中查找。通常，您应该将`——proto_path`标记设置为项目的根，并对所有导入使用完全限定名。
## 使用 proto2 消息类型
可以导入[proto2](https://developers.google.com/protocol-buffers/docs/proto)消息类型并在proto3消息中使用它们，反之亦然。但是，proto2枚举不能在proto3语法中直接使用(如果导入的proto2消息使用它们，也没关系)。
# 嵌套类型
您可以在其他消息类型中定义和使用消息类型，如下面的示例所示——这里的`Result`消息是在`SearchResponse`消息中定义的:
```
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```
如果希望在其父消息类型之外重用此消息类型，请将其称为`parent.type`:
```
message SomeOtherMessage {
  SearchResponse.Result result = 1;
}
```
你可以把信息嵌套得尽可能深:
```
message Outer {                  // Level 0
  message MiddleAA {  // Level 1
    message Inner {   // Level 2
      int64 ival = 1;
      bool  booly = 2;
    }
  }
  message MiddleBB {  // Level 1
    message Inner {   // Level 2
      int32 ival = 1;
      bool  booly = 2;
    }
  }
}
```
# 更新消息类型<a id=update></a>
如果现有的消息类型不再满足您的所有需求——例如，您希望消息格式有一个额外的字段——但是您仍然希望使用使用旧格式创建的代码，不要担心!在不破坏任何现有代码的情况下更新消息类型非常简单。只要记住以下规则:
* 不要更改任何现有字段的字段号
* 如果添加新字段，使用“旧”消息格式的代码序列化的任何消息仍然可以被新生成的代码解析。您应该记住这些元素的[默认值](#default)，以便新代码能够正确地与旧代码生成的消息交互。类似地，新代码创建的消息可以被旧代码解析:旧二进制文件在解析时完全忽略新字段。有关详细信息，请参阅[未知字段](#unknow)部分。
* 可以删除字段，只要在更新的消息类型中不再使用字段号。您可能希望重命名字段，比如添加前缀“OBSOLETE_”，或者将字段号设置为[reserved](#reserved-field)，这样.proto的未来用户就不会意外地重用该数字。
* `int32`、`uint32`、`int64`、`uint64`和`bool`都是兼容的——这意味着您可以将字段从其中一种类型更改为另一种类型，而不会破坏向前或向后兼容性。如果从不适合对应类型的连接解析一个数字，您将得到与在c++中转换该数字到该类型相同的效果(例如，如果64位数字被读取为`int32`，它将被截断为32位)。
* `sint32`和`sint64`相互兼容，但不兼容其他整数类型。
* `string`和`bytes`是兼容的，只要字节是有效的UTF-8。
* 如果`bytes`包含消息的编码版本，则嵌入式消息与`bytes`兼容。
* `fixed32`与`sfixed32`兼容，`fixed64`与`sfixed64`兼容。
* `enum`在连接格式方面与`int32`、`uint32`、`int64`和`uint64`兼容(注意，如果值不适合，将被截断)。但是要注意，当消息被反序列化时，客户端代码可能会以不同的方式对待它们:例如，未识别的`proto3` `enum`类型将保留在消息中，但在消息反序列化时如何表示则取决于语言。Int字段总是保留它们的值。
* 将单个值更改为**新**值的成员是安全和二进制兼容的。如果您确定一次没有多个代码集,将多个字段移动到一个新的`oneof`字段可能是安全的，将任何字段移动到现有的`oneof`字段是不安全的。
# 未知字段(Unknown Fields)<a id=unknow></a>
未知字段是格式良好的`protocol buffers`序列化数据，表示解析器无法识别的字段。例如，当一个旧二进制文件使用新字段解析由新二进制文件发送的数据时，这些新字段将成为旧二进制文件中的未知字段。  
最初，proto3消息在解析过程中总是丢弃未知字段，但是在3.5版本中，我们重新引入了未知字段的保存，以匹配proto2的行为。在3.5及更高版本中，解析过程中保留未知字段，并包含在序列化输出中。
# 任意(Any)
`Any`消息类型允许您将消息作为嵌入类型使用，而不需要它们的`.proto`定义。`Any`包含作为`bytes`的任意序列化消息，以及作为该消息类型的全局惟一标识符的URL。要使用`Any`类型，需要[导入](#other)`google/protobuf/any.proto`。
```
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```
给定消息类型的默认类型URL是`type.googleapis.com/packagename.messagename`.  
不同的语言实现将支持运行时库助手以类型安全的方式打包和解包任何值——例如，在Java中，任何类型都有特殊的`pack()`和`unpack()`访问器，而在c++中有`PackFrom()`和`UnpackTo()`方法:
```
// Storing an arbitrary message type in Any.
NetworkErrorDetails details = ...;
ErrorStatus status;
status.add_details()->PackFrom(details);

// Reading an arbitrary message from Any.
ErrorStatus status = ...;
for (const Any& detail : status.details()) {
  if (detail.Is<NetworkErrorDetails>()) {
    NetworkErrorDetails network_error;
    detail.UnpackTo(&network_error);
    ... processing network_error ...
  }
}
```
**目前，用于处理任何类型的运行时库都在开发中。**  
如果您已经熟悉[proto2语法](https://developers.google.com/protocol-buffers/docs/proto)，Any类型将替换[扩展](https://developers.google.com/protocol-buffers/docs/proto#extensions)。
# 其中一个(OneOf)
如果消息包含多个字段，并且最多只能同时设置一个字段，则可以使用`oneof`特性强制执行此行为并节省内存。除了所有字段共享内存外，`oneof`字段与常规字段类似，最多只能同时设置一个字段。设置其中的任何成员将自动清除所有其他成员。您可以使用特殊的`case()`或`which oneof()`方法检查`oneof`中的哪个值被设置(如果有)，这取决于您选择的语言。
## 使用 OneOf
要在.proto中定义oneof，需要使用oneof关键字和oneof名称，在本例中是test_oneo:
```
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```
然后将一个字段添加到一个定义中。可以添加除了`repeated`之外得任何类型的字段。在生成的代码中，其中一个字段具有与常规字段相同的getter和setter。您还可以获得一种特殊的方法来检查设置了其中一个值(如果有的话)。您可以在相关[API引用](https://developers.google.com/protocol-buffers/docs/reference/overview)中找到关于所选语言的API的更多信息。
## OneOf特征
* 设置一个字段将自动清除该字段的所有其他成员。因此，如果您设置了几个字段，那么只有*最后一个字段*仍然有值。
```
SampleMessage message;
message.set_name("name");
CHECK(message.has_name());
message.mutable_sub_message();   // Will clear name field.
CHECK(!message.has_name());
```
* 如果解析器在连接中遇到同一个`oneof`成员的多个成员，则解析后的消息中只使用最后看到的成员。
* A oneof cannot be `repeated`。
* 反射api为其中一个字段工作。
* 如果您使用c++，请确保您的代码不会导致内存崩溃。下面的示例代码将崩溃，因为通过调用`set_name()`方法，`sub_message`已经被删除。
```
SampleMessage message;
SubMessage* sub_message = message.mutable_sub_message();
message.set_name("name");      // Will delete sub_message
sub_message->set_...            // Crashes here
```
* 同样在c++中，如果您`Swap()`包含`oneof`的两条消息，每条消息的`oneof`设置的字段也会交换:在下面的示例中，msg1将有一个`sub_message`，msg2将有一个`name`。
```
SampleMessage msg1;
msg1.set_name("name");
SampleMessage msg2;
msg2.mutable_sub_message();
msg1.swap(&msg2);
CHECK(msg1.has_sub_message());
CHECK(msg2.has_name());
```
## 向后兼容性
添加或删除一个字段时要小心。如果检查`oneof`的值返回`None/NOT_SET`，则可能意味着尚未设置`oneof`，或者已将其设置为`oneof`的不同版本中的字段。由于无法知道连接中的未知字段是否属于其中的一个字段，所以无法区分它们。
**标签重用问题**
* **将字段移入或移出其中之一**:在序列化和解析消息之后，您可能会丢失一些信息(某些字段将被清除)。但是，您可以安全地将单个字段移动到**新**的`oneof`字段中并且如果已知只设置了一个字段，则可以移动多个字段。
* **删除一个字段并将其添加回来**:这可以在序列化和解析消息之后清除当前设置的oneof字段。
* **分割或合并`oneof`**:这与移动常规字段有类似的问题。
# 映射(Maps)
如果你想创建一个关联映射作为你的数据定义的一部分，`protocol buffers`提供了一个方便的快捷语法:
```
map<key_type, value_type> map_field = N;
```
`key_type`可以是任何整数或字符串类型(除了浮点类型和字节之外的任何[标量](#sca)类型)。注意：枚举类型不是有效的`key_type`。`value_type`可以是除另一个映射之外的任何类型。  
例如，如果您想创建一个项目映射，其中每个项目消息都与字符串类型的`key`相关联，您可以这样定义它:
```
map<string, Project> projects = 3;
```
* 映射字段不能是`repeated`
* 映射是无序的，因此，您不能依赖映射项的特定顺序。
* 当为`.proto`生成文本格式时，映射是按键排序的。数字键按数字排序。
* 当从连接解析或合并时，如果存在重复的映射键，则使用最后看到的键。从文本格式解析映射时，如果有重复键，解析可能失败。
* 如果您为map字段提供了一个键，但是没有值，那么序列化字段时的行为就依赖于语言。在c++、Java和Python中，按照类型的默认值进行序列化，而在其他语言中，没有任何东西被序列化。
## 向后兼容性
映射语法相当于下面的连接，因此不支持映射的`protocol buffers`实现仍然可以处理数据:
```
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```
任何支持映射的`protocol buffers`实现都必须生成和接受可以被上述定义接受的数据。
# 包(package)
您可以向`.proto`文件中添加一个可选的包说明符，以防止协议消息类型之间的名称冲突。
```
package foo.bar;
message Open { ... }
```
然后，您可以在定义消息类型的字段时使用包说明符:
```
message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}
```
包说明符影响生成代码的方式取决于您选择的语言:
* 在**c++**中，生成的类被包装在c++名称空间中。例如，Open将位于名称空间`foo::bar`中。
* 在**Java**中，包被用作Java包，除非您在`.proto`文件中显式地提供了一个选项`java_package`
* 在**Python**中，包指令被忽略，因为Python模块是根据它们在文件系统中的位置组织的。
* 在**Go**中，包被用作Go包的名称，除非您在`.proto`文件中显式地提供了一个选项`go_package`。
* 在**Ruby**中，生成的类封装在嵌套的Ruby名称空间中，转换为所需的Ruby大写样式(第一个字母大写;如果第一个字符不是字母，则PB_是前置的)。例如，`Open`将位于名称空间`Foo::Bar`中。
* 在**c#**中，包在转换为`PascalCase`后用作名称空间，除非您在`.proto`文件中显式地提供`option csharp_namespace`。例如，`Open`将位于名称空间`Foo.Bar`中。
## 包(package)和名(name)的区分
`protocol buffer`语言中的类型名称解析与c++类似:首先搜索最内层的作用域，然后是次内层作用域，等等，每个包都被认为是其父包的“内部”。开始的一个"."(例如，`.foo.bar.baz`)表示从最外面的范围开始。  
`protocol buffer`编译器通过解析导入的`.proto`文件来解析所有类型名。每种语言的代码生成器都知道如何引用该语言中的每种类型，即使它有不同的作用域规则。
# 定义服务
如果希望将消息类型与RPC(远程过程调用)系统一起使用，可以在`.proto`文件中定义RPC服务接口，`protocol buffer`编译器将用您选择的语言生成服务接口代码和存根。因此，例如，如果您希望使用接受`SearchRequest`并返回`SearchResponse`的方法定义RPC服务，您可以在`.proto`文件中定义它，如下所示:
```
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```
与`protocol buffers`一起使用的最直接的RPC系统是[gRPC](https://grpc.io/):在谷歌开发的语言和平台无关的开放源码RPC系统。gRPC特别适合使用`protocol buffers`，它允许您使用特殊的`protocol buffers`编译器插件直接从.proto文件生成相关RPC代码。  
如果您不想使用gRPC，也可以在自己的RPC实现中使用`protocol buffers`。您可以在[Proto2语言指南](https://developers.google.com/protocol-buffers/docs/proto#services)中了解更多。  
还有许多正在进行的第三方项目可以开发用于`protocol buffers`的RPC实现。有关到我们所知道的项目的链接列表，请参见[第三方插件wiki页面](https://github.com/protocolbuffers/protobuf/blob/master/docs/third_party.md)。
# JSON Mapping
Proto3支持JSON中的规范编码，使得在系统之间共享数据更加容易。下表按类型描述编码。  
如果json编码的数据中缺少一个值，或者它的值为null，那么在解析到`protocol buffers`时，它将被解释为适当的[默认值](#default)。如果字段在`protocol buffers`中具有默认值，则默认情况下json编码的数据将省略该值，以节省空间。实现可以提供在json编码的输出中使用默认值发出字段的选项。
|proto3|JSON|JSON example|notes|
|------|------|------|------|
|message|object|{"fooBar": v, "g": null, …}|生成JSON对象。消息字段名映射到小驼峰式并成为JSON对象键。如果指定了json_name字段选项，则指定的值将被用作键。解析器接受小驼峰式名称(或json_name选项指定的名称)和原始的proto字段名称。null是所有字段类型的可接受值，并被视为对应字段类型的默认值。|
|enum|string|"FOO_BAR"|使用proto中指定的枚举值的名称。解析器同时接受枚举名和整数值|
|map<K,V>|object|{"k": v, …}|所有键都被转换为字符串。|
|repeated V|array|[v, …]|null被接受为空列表[]。|
|bool|true, false|true, false||
|string|string|"Hello World!"||
|bytes|base64|string|"YWJjMTIzIT8kKiYoKSctPUB+"|JSON值是将数据编码为字符串，使用带填充的标准base64编码。标准或url安全的base64编码(带/不带paddings)都可以接受。|
|int32, fixed32, uint32|number|1, -10, 0|JSON值将是一个十进制数。可以接受数字或字符串。|
|int64, fixed64, uint64|string|"1", "-10"|JSON值将是一个十进制字符串。可以接受数字或字符串。|
|float, double|number|1.1, -10.0, 0, "NaN", "Infinity"|JSON值将是一个数字或一个特殊的字符串值“NaN”、“Infinity”和“-∞”。可以接受数字或字符串。指数表示法也被接受。|
|Any|object|{"@type": "url", "f": v, … }|如果Any包含一个具有特殊JSON映射的值，它将被转换为:{“@type”:xxx，“value”:yyy}。否则，该值将转换为JSON对象，并将插入“@type”字段以指示实际的数据类型。|
|Timestamp|string|"1972-01-01T10:00:20.021Z"|使用RFC 3339，其中生成的输出将始终为z规范化，并使用0、3、6或9个小数。除“Z”之外的补偿也可以接受。|
|Duration|string|"1.000340012s", "1s"|生成的输出总是包含0、3、6或9个小数，这取决于所需的精度，后面跟着后缀“s”。可以接受任何小数(也可以不接受)，只要它们符合毫微秒精度，并且需要后缀“s”。|
|Struct|object|{ … }|任何一个JSON对象。|
|Wrapper types|various types|2, "2", "foo", true, "true", null, 0, …|包装器在JSON中使用与封装的原语类型相同的表示，只是在数据转换和传输期间允许并保留null。|
|FieldMask|string|"f.fooBar,h"||
|ListValue|array|[foo, bar, …]||
|Value|value||任何JSON值|
|NullValue|null||JSON null|
## JSON 选项
一个proto3 JSON实现可以提供以下选项:
* **发出具有默认值的字段：**在proto3 JSON输出中，默认情况下会省略具有默认值的字段。一个实现可以提供一个选项来覆盖这个行为和带有默认值的输出字段。
* **忽略未知的字段：**在默认情况下，Proto3 JSON解析器应该拒绝未知字段，但是可以在解析中提供忽略未知字段的选项。
* **使用proto字段名而不是小驼峰式名称：**在默认情况下，proto3 JSON打印机应该将字段名称转换为小驼峰式，并将其用作JSON名称。实现可以提供一个选项来使用proto字段名作为JSON名称。需要Proto3 JSON解析器同时接受转换后的小驼峰式名称和proto字段名称。
* **将枚举值作为整数而不是字符串发出:**JSON输出中默认使用枚举值的名称。可以提供一个选项来代替枚举值的数值。
# 选项
`.proto`文件中的单个声明可以用许多选项进行注释。选项不会改变声明的总体含义，但可能会影响在特定上下文中处理声明的方式。可用选项的完整列表在`google/protobuf/descriptor.proto`中定义。  
有些选项是文件级别的选项，这意味着它们应该在顶级范围中编写，而不是在任何消息、枚举或服务定义中编写。有些选项是消息级别的选项，这意味着它们应该写在消息定义中。有些选项是字段级别的选项，这意味着它们应该写在字段定义中。还可以在枚举类型、枚举值、服务类型和服务方法上编写选项;但是，目前没有任何有用的选择。  
以下是一些最常用的选项:
* `java_package`(文件选项):希望为生成的Java类使用的包。如果`.proto`文件中没有给出显式`java_package`选项，那么默认情况下将使用proto包(在`.proto`文件中使用“package”关键字指定)。但是，proto包通常不能成为好的Java包，因为proto包不期望以反向域名开始。如果不生成Java代码，则此选项无效。
```
option java_package = "com.example.foo";
```
* `java_multiple_files` (file选项):在包级别定义顶级消息、枚举和服务，而不是在以`.proto`文件命名的外部类中定义。
```
option java_multiple_files = true;
```
* `java_outer_classname`(文件选项):要生成的最外层Java类的类名(以及文件名)。如果在.proto文件中没有指定显式java_outer_classname，则通过将.proto文件名转换为驼峰式(即foo_bar变为FooBar.java)来构造类名。如果不生成Java代码，则此选项无效。
```
option java_outer_classname = "Ponycopter";
```
* `optimize_for`(文件选项):可以设置为`SPEED`、`CODE_SIZE`或`LITE_RUNTIME`.这将通过以下方式影响c++和Java代码生成器(可能还有第三方生成器):
  * `SPEED`(默认):`protocol buffer`编译器将生成用于序列化、解析和对消息类型执行其他常见操作的代码。这段代码是高度优化的。
  * `CODE_SIZE`:`protocol buffer`编译器将生成最少的类，并依赖于共享的、基于反射的代码来实现序列化、解析和各种其他操作。因此，生成的代码将比`SPEED`小得多，但是操作将更慢。类仍然会实现与在SPEED模式下完全相同的公共API。这种模式在包含大量`.proto`文件的应用程序中非常有用，而且不需要所有文件都快得令人眼花缭乱。
  * `LITE_RUNTIME`:`protocol buffer`编译器将生成仅依赖于“lite”运行时库(`libprotobuf-lite`，而不是`libprotobuf`)的类。lite运行时比完整库小得多(大约小了一个数量级)，但是省略了某些特性，比如描述符和反射。这对于在手机等受限平台上运行的应用程序尤其有用。编译器仍然会生成所有方法的快速实现，就像它在`SPEED`模式下所做的那样。生成的类只会在每种语言中实现MessageLite接口，它只提供完整`Message`接口方法的一个子集。
```
option optimize_for = CODE_SIZE;
```
* `cc_enable_arenas`(文件选项):为c++生成的代码启用[竞技场分配](https://developers.google.com/protocol-buffers/docs/reference/arenas)。
* `objc_class_prefix`(文件选项):设置Objective-C类前缀，该前缀位于所有Objective-C生成的类和由此`.proto`生成的枚举之前。没有默认。你应该使用[苹果推荐](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Conventions/Conventions.html#//apple_ref/doc/uid/TP40011210-CH10-SW4)的3-5个大写字母之间的前缀。请注意，所有两个字母前缀都是由Apple保留的。
* `deprecated`(字段选项):果设置为true，则表示该字段已弃用，不应由新代码使用。在大多数语言中，这没有实际效果。在Java中，这变成了`@Deprecated`注释。将来，其他特定于语言的代码生成器可能会在字段的访问器上生成弃用注释，这反过来会在编译试图使用字段的代码时发出警告。如果任何人都不使用该字段，并且您希望阻止新用户使用该字段，请考虑用[保留](#reserved-field)语句替换该字段声明。
```
int32 old_field = 6 [deprecated=true];
```
## 自定义选项
`protocol buffer`还允许您定义和使用自己的选项。这是一个大多数人不需要的**高级功能**。如果您确实认为需要创建自己的选项，请参阅[Proto2语言指南](https://developers.google.com/protocol-buffers/docs/proto.html#customoptions)了解详细信息。注意，创建自定义选项使用[扩展](https://developers.google.com/protocol-buffers/docs/proto.html#extensions)，而扩展只允许在proto3中使用自定义选项。
# 生成类
要生成Java、Python、c++、Go、Ruby、Objective-C或c#代码，您需要处理.proto文件中定义的消息类型，您需要在.proto上运行`protocol buffer`编译器原型。如果还没有安装编译器，请[下载这个包](https://developers.google.com/protocol-buffers/docs/downloads.html)并按照README中的说明操作。对于Go，您还需要为编译器安装一个特殊的代码生成器插件:您可以在GitHub上的[golang/protobuf](https://github.com/golang/protobuf/)存储库中找到这个插件和安装说明。  
`Protocol`编译器调用方式如下:
```
protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR --python_out=DST_DIR --go_out=DST_DIR --ruby_out=DST_DIR --objc_out=DST_DIR --csharp_out=DST_DIR path/to/file.proto
```
* `IMPORT_PATH`指定在解析导入指令时查找`.proto`文件的目录。如果省略，则使用当前目录。可以通过多次传递——`proto_path`选项指定多个导入目录;他们将按顺序被搜查。`-I=IMPORT_PATH can be used as a short form of --proto_path.`
* 您可以提供一个或多个输出指令:
  * `——cpp_out`在`DST_DIR`中生成c++代码。有关更多信息，请参见[c++生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/cpp-generated)。
  * `——java_out`在`DST_DIR`中生成Java代码。有关更多信息，请参阅[Java生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/java-generated)。
  * `——python_out`在`DST_DIR`中生成Python代码。有关更多信息，请参见[Python生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/python-generated)。
  * `——go_out`在`DST_DIR`中生成Go代码。有关更多信息，请参阅[Go生成的代码引用](https://developers.google.com/protocol-buffers/docs/reference/go-generated)。
  * `——ruby_out`在`DST_DIR`中生成Ruby代码。Ruby生成的代码引用即将到来!
  * `——objc_out`在`DST_DIR`中生成Objective-C代码。有关更多信息，请参见[Objective-C生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/objective-c-generated)。
  * `——csharp_out`在`DST_DIR`中生成c#代码。有关更多信息，请参见[c#生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/csharp-generated)。
  * `——php_out`在`DST_DIR`中生成PHP代码。有关更多信息，请参见[PHP生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/php-generated)。
  为了方便，如果`DST_DIR`以`.zip`或`.jar`结尾，编译器将把输出写入具有给定名称的单个zip格式存档文件。`.JAR`输出还将根据Java JAR规范的要求提供一个清单文件。注意，如果输出存档已经存在，则会覆盖它;编译器不够智能，无法将文件添加到现有的存档中。
* 您必须提供一个或多个`.proto`文件作为输入。可以同时指定多个`.proto`文件。虽然这些文件是相对于当前目录命名的，但是每个文件必须驻留在`IMPORT_PATH`之一中，以便编译器能够确定其规范名称。
