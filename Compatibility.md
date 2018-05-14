# 兼容性
本章提供了有关[版本化](https://git.garena.com/shopee/space/api-design-guide/blob/36c7960d6005269ea8d6de1ce0d1db632a589ac2/Versioning.md)控制部分中给出的破坏和保持兼容性修改的详细说明

一个变化是不是破坏的（incompatible）有时候是很难弄清楚的，所以这里只是提供一些指示性的指导而不是列出每一个有可能是破坏性的更改。

下面列出的这些规则只涉及客户端兼容性，默认API生产者会意识到他们在部署方面的需求，包括实施细节的变化。

一般的目的是服务端新的`MINOR`或者`PATCH`更改不应该对客户端造成破坏。这里所说的破坏包括以下几个方面：

* 源代码兼容性：基于1.0编写的代码在变成基于1.1后不能编译
* 二进制兼容性：基于1.0编写的代码不能在1.1客户端库中进行link和run（具体的细节依赖客户端，不同情况有不同变化）
* 协议兼容性：基于1.0编写的代码不能和1.1版本的服务端通信
* 语法兼容性：所有组件都能运行但产生意想不到的结果

简而言之：在服务端进行`MINOR`版本更新后，旧的客户端还可以与之通信，并且客户端如果也想进行一次`MINOR`升级（使用服务器`MINOR`升级后提供的新的属性），他们很容易就能做到。

由于客户端的代码包括自动生成的代码和手写的代码两部分，所以除了理论上的基于协议的考虑因素外，我们还要考虑一些实际的情况。当你在考虑对某一些地方进行更改的时候，如果有可能，请生成新版本的客户端代码来对你的改变进行测试以确保其还能通过测试。


下面的讨论会将proto消息（messages）分成三个类别：
* 请求消息（Request messages）（例如`GetBookRequest`）
* 响应消息（Response messages）（例如`ListBooksResponse`）
* 资源消息（Resource messages）（例如`Book`，以及那些被其他资源消息用到的任何消息）

对于这些不同类别的消息，应该使用不同的规则，因为一般来说请求消息只会从客户端发给服务端，响应消息只会从服务端发给客户端的，但资源信息通常是双向的。尤其是可被修改的资源需要根据`读取/修改/写入`的循环来考虑。

## 向后兼容的改变
### 为API服务添加一个API接口
从protocol的角度来说，这种改变总是安全的。唯一一个需要注意的情况是客户端可能在自己实现的代码里面使用了你的新API接口名称。不过如果你的新API接口是和现存的代码完全正交的话这种情况基本不可能发生。如果新的接口是现存接口的一个简化版本的话，这种情况很有可能会导致冲突发生。

### 为API接口添加一个方法
除非你要添加一个与客户端生成库中的某个方法冲突的方法，否则这种修改没有问题。

举个例子：如果你已经有一个叫做`GetFoo`的方法，C##的生成器会据此生成两个方法，一个叫`GetFoo`，另外一个叫`GetFooSync`，在这种情况下，如果你再给这个接口添加一个`GetFooSync`方法就会和客户端的代码发生冲突。

### 为方法添加一个HTTP绑定
如果新的绑定没有引入歧义，让客户端能够响应一个其之前拒绝的请求一般没有问题。新的绑定可能通过将已有的操作应用到一个新的资源名称来完成。

### 为请求消息添加一个字段
如果新版本的服务端对待那些没有指明新字段的客户端请求像旧版本一样，那么为请求消息添加一个新的字段是没有破坏性的改变。

一个很明显的新增的具有破坏性的字段就是分页（pagination）:如果API v1版本对某个集合原来不支持分页的话，就不应该在v1.1版本里面引入这个字段，除非`page_size`的默认值是应该设置为无限的（infinite），不过这很明显是不好的设计想法。可是如果不这么做，v1.0的客户端就会从服务端里面收到已经被截取过的数据，可是他们还完全不知情。

### 为响应消息添加一个字段
在不改变其他响应字段行为的前提下，非资源响应信息（例如 `ListBooksResponse`）可以被扩展而不会破坏客户端的代码。即使会导致冗余，任何在旧的响应消息中的字段也应该存在于新的响应中并保持它原来的语义。

例如，如果在v1.0的响应里面有个叫做`contained_duplicates`的布尔值字段，这个字段用来表明某些结果因为重复而被省略了，我们可能在v1.1版本里提供一个新的`duplicate_count`字段来为用户提供更多的信息，不过我们必须继续保留`contained_duplicates`字段，虽然从1.1版本的角度来看这会产生冗余。

### 为枚举类型添加一个值
只用在请求消息里面的枚举字段可以添加任意新元素来进行扩展。例如，在使用[资源视图模式](https://cloud.google.com/apis/design/design_patterns#resource_view)（Resource View）时，你可以添加一个新的视图到`MINOR`版本里面。客户端永远也不会接收到这个枚举值，所以他们不需要关心。

对于资源和响应消息，默认的假设是客户端应该处理它们不知道的枚举值。虽然这样，API提供者也应该意识到编写程序去处理新的枚举值是一件很难的事情，所以他们应该在文档里面写明白当客户端遇到一个未知的枚举值时应该怎么处理。

proto3允许客户端接收它们不关心的值并且重新序列化消息时会保持值不变，这样就不会打破`读取/修改/写入`循环的兼容性。JSON格式允许发送数值，其中该值的“名称”是未知的，但是服务端通常不会知道客户端是否真正知道特定值。因此JSON客户端可能知道它们已经收到了之前不知道的值，但他们只会看到名称或数字而不会两个都可以看到。在`读取/修改/写入`循环中将相同的值返回给服务端而不应该修改这个值，因为服务端会理解这两种形式。

### 添加只供输出（output-only）的资源字段 
可以添加仅由服务端提供的资源实体字段。服务端可以验证客户请求中的任意值是否是有效的，但是就算该值被省略了，这个请求也一定不能失败。

## 不向后兼容的更改
### 删除或者重命名一个服务，字段或者枚举值

从根本上说，如果客户端使用了要更改的值，那么删除和重命名就是一个破坏性的改变，一定要进行一个`MAJOR`版本的升级。对于某些语言（例如C#和Java），那些使用了旧名称的代码将会在编译的时候发生错误，对于其他语言则可能导致运行错误或者数据丢失。协议格式的兼容性在这里是无关紧要的。

### 更改HTTP绑定

这里的修改实际指`删除`和`添加`。例如，你想要支持PATCH操作，但已发布的版本支持PUT操作，或者已经使用了错误的自定义动词，你可以添加新的绑定，但是一定不要移除旧的，因为这会和删除某个服务一样破坏兼容性。

### 更改某个字段的类型
即使新的类型是协议兼容的，客户端自动生成的代码还是会因为字段类型的改变而发生改变，对于那种静态编译语言可能会导致代码在编译的时候发生错误，所以这种情况一定要进行一次`MAJOR`版本的升级。

### 更改资源名称格式
资源的名称一定不能被改变，这同时也就意味着集合的名称也不能被改变。

不像其他大多数破坏兼容性的修改，这还会影响`MAJOR`版本号：如果客户端期望使用v2.0的API来访问在v1.0中创建的资源（反过来也一样），则应该在两个版本中使用相同的资源名称。

有效的资源名称集也不应该改变，原因如下：

* 如果有效资源集变得更小，先前能够成功的请求可能会变失败。
* 如果有效资源集变得更大，客户端基于先前的文档做的假设将会失效。客户端很可能在其他地方保存了资源名，并且对字符集和名字的长度敏感。或者，客户端可能会执行自己的资源名称验证来保持与文档一致。（举个例子，[亚马逊在开始允许更长EC2资源IDs的时候给予了客户端许多警告以及允许他们在一段时间内完成迁移](https://aws.amazon.com/blogs/aws/theyre-here-longer-ec2-resource-ids-now-available/)。）

请注意，此类更改只在proto文档里面可见，所以在审查破坏性事故时（CL for breakage），只是查看非注释的改变是不够的。

### 修改已有请求的可见性（visible behavior）
客户端总是依赖API的行为和语义，`即使没有明确支持或将这个行为写入文档`, 所以在大多数情况下修改API的行为和语义在客户端看来都是破坏性的。如果某行为不是加密隐藏的，你就应该假设用户已经依赖它了。

加密分页的token是个好的解决这个问题的办法（即使数据无关紧要），可以防止用户创建自己的token和当token行为发生变化时可能带来的不兼容性。

### 在HTTP定义中改变URL格式
除了上面说的资源名称发生改变还要考虑另外两种改变：

* 自定义方法名称：虽然它不是资源名称的一部分，不过它是REST客户端发起的URL请求的一部分。所以改变自定义方法名称虽然不会对gRPC的客户端造成影响，我们还是要考虑这个改变对REST客户端造成的影响，因为对于公共API来说，我们都要假设它们会有相应的REST客户端。
* 资源参数名：从 `v1/shelves/{shelf}/books/{book} `到 `v1/shelves/{shelf_id}/books/{book_id} `的修改不会影响替代的资源名称，但是会对自动生成的代码造成影响。

### 在资源消息中添加读/写字段

客户端会经常执行`读取/修改/写入`的操作。大多数客户端不会为它们不知道的字段赋值，特别是在proto3中不支持。你可以指明消息类型（而不是原始类型）中缺失的字段表示这个字段在更新时不会被修改，但这样使要明确删除这样的字段变的困难。原始类型（包括string和bytes）不能简单地使用这种方法，因为在proto3中，明确地设置 int32 的值为 0 和不对它设置值是没有区别的。

使用字段掩码来进行所有更新操作不会有问题，因为客户端不会隐式覆盖其不知道的字段。然而这不是一个正常的决定，因为大部分 API 允许全部资源被更新。