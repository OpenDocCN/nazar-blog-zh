#  RFC 7168:茶流出设备的超文本咖啡壶控制协议

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/RFC-7168:-Hypertext-Coffee-Pot-Control-Protocol-for-Tea-Efflux-Appliances>

*本文件由 [RFC 编辑](http://web.archive.org/web/20220810161352/http://www.rfc-editor.org/)于 2014 年 4 月 1 日发布。*

*独立投稿
征求意见: [7168](http://web.archive.org/web/20220810161352/http://www.rfc-editor.org/rfc/rfc7168.txt)
更新: [2324](http://web.archive.org/web/20220810161352/http://www.rfc-editor.org/rfc/rfc2324.txt)
类别:信息
刊号:2070-1721*

## 摘要

超文本咖啡壶控制协议(HTCPCP)规范不允许茶的冲泡，不管其种类和复杂性如何。这篇论文概述了 HTCPCP 的扩展，允许壶提供网络化的泡茶设备。

## 此备忘录的状态

本文档不是互联网标准跟踪规范；它的发布是为了提供信息。

这是对 RFC 系列的贡献，独立于任何其他 RFC 流。RFC 编辑器已自行决定发布本文档，并且未对其实现或部署价值做出任何声明。RFC 编辑批准出版的文档不属于任何级别的互联网标准；参见 RFC [5741](http://web.archive.org/web/20220810161352/http://www.rfc-editor.org/rfc/rfc5741.txt) 的第 2 节。

有关本文档的当前状态、任何勘误表以及如何提供反馈的信息可在[http://www.rfc-editor.org/info/rfc7168](http://web.archive.org/web/20220810161352/http://www.rfc-editor.org/info/rfc7168)获得。

## 版权声明

版权所有(c) 2014 IETF Trust 和文档作者。保留所有权利。

本文件受 BCP 78 和 IETF Trust 有关 IETF 文件的法律规定([http://trustee.ietf.org/license-info](http://web.archive.org/web/20220810161352/http://trustee.ietf.org/license-info))的约束，这些法律规定在本文件发布之日有效。请仔细阅读这些文档，因为它们描述了您对本文档的权利和限制。从本文档中提取的代码组件必须包括信托法律条款第 4.e 节中所述的简化 BSD 许可证文本，并按照简化 BSD 许可证中所述提供无担保。

## 1.介绍

正如在[超文本咖啡壶控制协议](http://web.archive.org/web/20220810161352/http://www.rfc-editor.org/rfc/rfc2324.txt)中所提到的，咖啡作为一种巧妙酿造的含咖啡因饮料而享誉全球，但咖啡与许多其他基于植物材料过滤的各种制剂具有相同的品质。最重要的是，其中有一类是基于从茶树上制备的叶子过滤水的酿造:茶属的血统和历史将不作为本文的一部分叙述，但有证据表明茶叶的生产在几千年前就存在了。

HTCPCP 在解决像茶这样古老的饮料的网络化生产方面的不足是值得注意的:事实上，对网络化茶壶的唯一规定是它们不响应生产咖啡的请求，这虽然非常合理，但不允许与茶壶进行通信以实现其预期目的。

本文详细说明了 HTCPCP 的扩展，以允许与联网的茶叶生产设备和茶壶进行通信。对这里指定的协议的添加允许控制所有能够制作最受欢迎的含咖啡因的热饮料的设备所必需的请求和响应。

### 1.1.术语

本文件中的关键词*必须*、*不得*、*必需*、*应当*、*不得*、*应当*、*不应*、*推荐*、*可能*、*可选*按照 RFC [2119](http://web.archive.org/web/20220810161352/http://www.rfc-editor.org/rfc/rfc2119.txt) 中的描述进行解释。

## 2.HTCPCP-茶叶协议增补

茶叶扩展到 HTCPCP 适应某些 HTCPCP 方法的操作。

### 2.1.`BREW`和`POST`方法

如基本 HTCPCP 规范中所述，通过发送`BREW`请求来控制可泡茶的壶。`POST`请求被同等对待，但仍被否决。然而，茶的生产不同于咖啡，因为在茶被冲泡之前，通常会为顾客提供茶的选择。为此，接收到内容类型`message/teapot` *的`BREW`消息的可泡茶的壶必须*根据 URI 请求做出响应，如下所示。

#### 2.1.1.`/` URI

对于 URI `/`，酿造将不会开始。相反，必须发送 RFC [2295](http://web.archive.org/web/20220810161352/http://www.rfc-editor.org/rfc/rfc2295.txt) *中定义的`Alternates`标题，其中包含可用的茶叶袋和/或茶叶品种作为条目。这种响应的一个例子如下:*

```
      Alternates: {"/darjeeling" {type message/teapot}},
                  {"/earl-grey" {type message/teapot}},
                  {"/peppermint" {type message/teapot}}

```

以下示例展示了符合 HTCPCP 基本规范的可泡茶壶的互操作性的可能性:

```
      Alternates: {"/" {type message/coffeepot}},
                  {"/pot-0/darjeeling" {type message/teapot}},
                  {"/pot-0/earl-grey" {type message/teapot}},
                  {"/pot-1/peppermint" {type message/teapot}}

```

支持 TEA 的 HTCPCP 客户端*必须*检查由`BREW`请求返回的`Alternates`报头的内容，并为`message/teapot`类型的后续请求提供一个特定的 URI。

对具有`message/coffeepot` *的`Content-Type`标题的`/` URI 的请求*也应该用上述格式的`Alternates`标题来响应，以允许有喝茶能力的客户有机会向用户呈现茶的选择，如果最初请求的是劣质的含咖啡因饮料的话。

#### 2.1.2.品种特异性 URIs

当收到对特定品种茶叶的`BREW`请求时，可泡茶的壶遵循基本 HTCPCP 规范。壶*要*按照每个品种给出的酿造浓度建议，达到这个浓度就停止酿造；建议通过检测当前由壶冲泡的饮料的不透明度来测量浓度。

支持 TEA 的客户端*应该*通过发送包含`stop`的实体主体的`BREW`请求来指示冲泡的结束；锅*可能*继续酿造超过推荐浓度，直到收到该浓度。如果`stop`请求不是由客户端发送的，这可能导致冲泡壶中茶与水的比例的状态反转，这可能被一些壶报告为负强度。

如果在达到推荐浓度之前收到带有包含`stop`的实体的`BREW`命令，壶*必须*中止冲泡，并以较低的浓度供应所得饮料。使用此超驰功能时，找到首选的饮料浓度是茶壶收到`start`请求和随后的`stop`请求之间的时间函数。客户*应该*做好多次尝试以达到首选强度的准备。

### 2.2.修改的标题字段

HTCPCP-TEA 修改了基本 HTCPCP 规范中一个报头字段的定义。

#### 2.2.1.`Accept-Additions`标题字段

据观察，一些混合茶的使用者偶尔会偏爱用蔗糖乳液和少量水冲泡的茶。考虑到这种情况，基本 HTCPCP 规范中定义的`Accept-Additions`报头字段被更新为允许以下选项:

```
      addition-type   = ( "*"
                        | milk-type
                        | syrup-type
                        | sweetener-type
                        | spice-type
                        | alcohol-type
                        | sugar-type
                        ) *( ";" parameter )
      sugar-type      = ( "Sugar" | "Xylitol" | "Stevia" )

```

实现者应该意识到过度使用`Sugar`添加可能会导致`BREW`请求超过传输层允许的段大小，从而导致碎片和酝酿延迟。

### 2.3.响应代码

HTCPCP-TEA 使用正常的 HTTP 错误代码和那些在基本 HTCPCP 规范中定义的错误代码。

#### 2.3.1.300 多种选择

对`/` URI 的`BREW`请求，如第 2.1.1 节中所定义的，将返回一个`Alternates`头，指示可供冲泡的茶叶品种的 URIs。*建议*使用状态代码 300 来提供该响应，以表明酿造尚未开始，进一步的选项必须由客户选择。

#### 2.3.2.403 禁止

如果服务认为所请求的添加组合违背了饮者对所述品种的共识的敏感性，则实现`Accept-Additions`标题字段*的服务可以*返回 403 状态码，用于给定品种茶的`BREW`请求。

收集和整理各种食品添加剂最可行组合的共识指标的方法不在本文件的讨论范围内。

#### 2.3.3.418 我是茶壶

不用于冲泡咖啡的可泡茶的壶可以返回状态代码 503，表示暂时没有咖啡，或者返回基本 HTCPCP 规范中定义的代码 418，表示壶是茶壶的更永久的指示。

## 3.`message/teapot`媒体类型

为了区分去往支持 TEA 的 HTCPCP 服务的消息和符合基本 HTCPCP 规范的 pots，本文档定义了一种新的 MIME 媒体类型。如果要请求茶，发送到可泡茶的壶*的`POST`或`BREW`请求的`Content-Type`报头必须*为`message/teapot`。

## 4.环境因素

如第 2.1 节所述，一个带有`message/teapot`的`Content-Type`报头字段的`BREW`请求将导致一个`Alternates`报头与响应一起被发送，壶将不会被冲泡。然而，如果`BREW`请求的`Content-Type`为`message/coffeepot`，并且壶能够冲泡咖啡，那么服务的行为将退回到基本 HTCPCP 规范，并且壶将被冲泡。

如果服务器在冲泡开始时返回的实体包含一个符合 TEA 标准的`Alternates`头，指示`message/coffeepot`，而客户端不需要咖啡，那么客户端*应该*发送一个带有包含`stop`的实体主体的`BREW`请求。这将导致浪费咖啡；这是否被视为一件坏事是用户定义的。

能够喝茶的客户可以通过首先请求类型`message/teapot`的`BREW`，然后允许选择可用的饮料来防止这种浪费。

## 5.安全考虑

与基本的 HTCPCP 规范一样，大多数能泡茶的壶都是通过使用电气元件来加热水的，因此不会靠近火。因此，与这些 pot 进行通信不需要防火墙。

此扩展支持与燃烧罐的通信，但是，这可能需要保温和控制策略。应注意不要将燃煤锅和电热水壶连接到同一个网络，以防止锅将网络上的任何水壶称为黑暗或烟雾驱动。

## 6.承认

如果没有基本规范，对网络化饮料生产的研究，对 HTCPCP 规范的扩展是不可能的。在这种情况下，作者希望感谢 Larry Masinter 在开发咖啡壶通信领先协议方面所做的出色工作。

非常感谢[凯文·沃特森](http://web.archive.org/web/20220810161352/http://phpro.org/)和[皮特·戴维斯](http://web.archive.org/web/20220810161352/http://pete-davis.info/)，他们在本文件起草期间提供了指导和建议。

*文章日期:2014 年 4 月 1 日*