# Telebot
>"我从未想过创建Telegram机器人可以如此"性感"！"

[![GoDoc](https://godoc.org/gopkg.in/telebot.v3?status.svg)](https://godoc.org/gopkg.in/telebot.v3)
[![GitHub Actions](https://github.com/tucnak/telebot/actions/workflows/go.yml/badge.svg)](https://github.com/tucnak/telebot/actions)
[![codecov.io](https://codecov.io/gh/tucnak/telebot/coverage.svg?branch=v3)](https://codecov.io/gh/tucnak/telebot)
[![Discuss on Telegram](https://img.shields.io/badge/telegram-discuss-0088cc.svg)](https://t.me/go_telebot)

```bash
go get -u gopkg.in/telebot.v3
```

* [概览](#概览)
* [入门](#入门)
	- [上下文](#上下文)
	- [中间件](#中间件)
	- [轮询器](#轮询器)
	- [命令](#命令)
	- [文件](#文件)
	- [可发送对象](#可发送对象)
	- [可编辑对象](#可编辑对象)
	- [键盘](#键盘)
	- [内联模式](#内联模式)
* [贡献](#贡献)
* [许可证](#许可证)

# 概览
Telebot是一个用于[Telegram Bot API](https://core.telegram.org/bots/api)的机器人框架。该包提供了一种最佳的API，用于命令路由、内联查询请求和键盘，以及回调。实际上，我更进一步，选择专注于API的美观和性能，而不是创建一个1:1的API封装器。Telebot的一些优点包括：

* 非常简洁的API
* 命令路由
* 中间件
* 透明的文件API
* 高效率的机器人回调


Telebot API的所有方法都非常容易记忆和使用。此外，考虑到高负载的情况，Telebot是一个高性能的解决方案。我将测试和基准测试最常用的操作，并在必要时进行优化，而不会影响API的质量。

# 入门
让我们看一下最简单的Telebot设置：

```go
package main

import (
	"log"
	"os"
	"time"

	tele "gopkg.in/telebot.v3"
)

func main() {
	pref := tele.Settings{
		Token:  os.Getenv("TOKEN"),
		Poller: &tele.LongPoller{Timeout: 10 * time.Second},
	}

	b, err := tele.NewBot(pref)
	if err != nil {
		log.Fatal(err)
		return
	}

	b.Handle("/hello", func(c tele.Context) error {
		return c.Send("Hello!")
	})

	b.Start()
}

```

简单吧？Telebot的路由系统负责将更新传递到其端点，因此为了处理任何有意义的事件，你只需将函数插入到Telebot提供的端点之一即可。你可以在此处找到完整列表：[这里](https://godoc.org/gopkg.in/telebot.v3#pkg-constants).

有几十个支持的端点（参见软件包常量）。如果你希望看到某些端点或端点的实现，请告诉我。该系统是完全可扩展的，因此我可以在不破坏向后兼容性的情况下引入它们。

## 上下文
上下文是一种特殊的类型，它封装了一个庞大的更新结构，并表示当前事件的上下文。它提供了几个辅助函数，允许获取例如发送此更新的聊天的函数，无论这是什么类型的更新。

```go
b.Handle(tele.OnText, func(c tele.Context) error {
	// 所有未被现有处理程序捕获的文本消息

	var (
		user = c.Sender()
		text = c.Text()
	)

	// 仅在需要结果时使用完整的机器人函数：
	msg, err := b.Send(user, text)
	if err != nil {
		return err
	}

	// 相反，最好使用上下文简写：
	return c.Send(text)
})

b.Handle(tele.OnChannelPost, func(c tele.Context) error {
	// 仅限频道帖子。
	msg := c.Message()
})

b.Handle(tele.OnPhoto, func(c tele.Context) error {
	// 仅限照片。
	photo := c.Message().Photo
})

b.Handle(tele.OnQuery, func(c tele.Context) error {
	// 输入的内联查询。
	return c.Answer(...)
})
```

## 中间件
Telebot具有一种简单且易于识别的方法来设置中间件-具有对`Context`访问权限的链接函数，在处理程序执行之前被调用。

导入`middleware`包以获得一些基本的开箱即用的中间件实现：
```go
import "gopkg.in/telebot.v3/middleware"
```

```go
// 全局范围的中间件：
b.Use(middleware.Logger())
b.Use(middleware.AutoRespond())

// 组范围的中间件：
adminOnly := b.Group()
adminOnly.Use(middleware.Whitelist(adminIDs...))
adminOnly.Handle("/ban", onBan)
adminOnly.Handle("/kick", onKick)

// 处理程序范围的中间件：
b.Handle(tele.OnText, onText, middleware.IgnoreVia())
```

Custom middleware example:
```go
// AutoResponder自动回复每个回调更新。
func AutoResponder(next tele.HandlerFunc) tele.HandlerFunc {
	return func(c tele.Context) error {
		if c.Callback() != nil {
			defer c.Respond()
		}
		return next(c) // 继续执行链
	}
}
```

## 轮询器
Telebot并不真正关心您如何提供传入的更新，只要您使用轮询器设置它，或者对每个更新调用`ProcessUpdate`方法：

```go
// Poller是更新的提供者。
//
// 所有轮询器都必须实现Poll()方法，该方法接受机器人指针和订阅通道，并立即开始同步轮询。
type Poller interface {
	// Poll应该接受机器人对象、订阅通道，并立即开始轮询更新。
	//
	// 轮询器必须不断监听停止信号，并在完成轮询时关闭它。
	Poll(b *Bot, updates chan Update, stop chan struct{})
}
```

## 命令
在处理命令时，Telebot支持直接命令（`/command`）和类似组的语法（`/command@botname`），即使隐私模式关闭，它也永远不会将消息发送给其他机器人。

为了简化深度链接，Telebot还提取有效载荷：
```go
// 命令: /start <PAYLOAD>
b.Handle("/start", func(c tele.Context) error {
	fmt.Println(c.Message().Payload) // <PAYLOAD>
})
```

对于多个参数，请使用：
```go
// 命令: /tags <tag1> <tag2> <...>
b.Handle("/tags", func(c tele.Context) error {
	tags := c.Args() // 由空格分隔的参数列表
	for _, tag := range tags {
		// 遍历传递的参数
	}
})
```

## 文件
>Telegram允许最大50MB的文件。

Telebot允许在机器人的范围内同时上传（从磁盘或通过URL）和下载（从Telegram）文件。此外，使用从磁盘创建的文件发送任何类型的媒体自动将文件上传到Telegram：
```go
a := &tele.Audio{File: tele.FromDisk("file.ogg")}

fmt.Println(a.OnDisk()) // true
fmt.Println(a.InCloud()) // false

// 将从磁盘上传文件并发送给收件人
b.Send(recipient, a)

// 下次您发送这个相同的*Audio，Telebot不会重新上传相同的文件，而是使用其Telegram FileID
b.Send(otherRecipient, a)

fmt.Println(a.OnDisk()) // true
fmt.Println(a.InCloud()) // true
fmt.Println(a.FileID) // <Telegram file ID>
```
您可能希望保存某些文件，以避免重新上传。随意将它们编组为任何格式，`File`仅包含公共字段，因此不会丢失任何数据。

## 可发送对象
`Send`无疑是Telebot中最重要的方法。`Send()`接受一个`Recipient`（可以是用户、群组或频道）和一个`Sendable`。除了Telebot提供的媒体类型（`Photo`、`Audio`、`Video`等）之外，其他类型也是可发送的。如果您创建了自己的复合类型，并且它们满足`Sendable`接口，Telebot将能够发送它们。

```go
// Sendable 是任何可以发送自身的对象。
//
// 这非常酷，因为它允许机器人为复杂的媒体或跨多个消息的聊天对象实现自定义Sendables。
type Sendable interface {
	Send(*Bot, Recipient, *SendOptions) (*Message, error)
}
```

目前唯一不适用于`Send()`的类型是`Album`，这是有原因的。出于向后兼容性的考虑，相册是最近才添加的，因此它们对于实现兼容性来说有些怪异。实际上，可以发送相册，但无法接收。相反，Telegram会返回一个`[]Message`，其中每个媒体对象在相册中对应一个消息：

```go
p := &tele.Photo{File: tele.FromDisk("chicken.jpg")}
v := &tele.Video{File: tele.FromURL("http://video.mp4")}

msgs, err := b.SendAlbum(user, tele.Album{p, v})
```

### 发送选项

发送选项是您可以作为可选参数（跟在接收者和文本/媒体后面）传递给`Send()`、`Edit()`等方法的对象和标志。最重要的是`SendOptions`，它允许您控制Telegram支持的消息的所有属性。唯一的缺点是在某些情况下它不太方便使用，因此`Send()`支持多个简写形式：
```go
// 常规发送选项
b.Send(user, "text", &tele.SendOptions{
	// ...
})

// ReplyMarkup是SendOptions的一部分，
// 但通常它是您唯一需要的选项
b.Send(user, "text", &tele.ReplyMarkup{
	// ...
})

// 标志：无通知和无网络链接预览
b.Send(user, "text", tele.Silent, tele.NoPreview)
```

支持的选项标志的完整列表可以在[这里找到](https://pkg.go.dev/gopkg.in/telebot.v3#Option)。

## 可编辑对象

如果要编辑某个现有消息，您实际上不需要存储原始的`*Message`对象。实际上，在编辑时，Telegram仅需要`chat_id`和`message_id`。因此，您不需要完整的消息。此外，您可能希望将某些消息的引用存储在数据库中，以避免重新上传。为此，我认为任何Go结构体都可以作为Telegram消息进行编辑，实现`Editable`接口：
```go
// Editable 是提供“消息签名”的所有对象的接口，它是一个由32位消息ID和64位聊天ID组成的对，这两个对于编辑操作都是必需的。
//
// 使用情况：为待编辑的消息创建的数据库模型结构体，例如：msg_id、chat_id两列，可以很容易地实现 MessageSig()，使得存储的消息实例可编辑。
type Editable interface {
	// MessageSig 是“消息签名”。
	//
	// 对于内联消息，请返回 chatID = 0。
	MessageSig() (messageID int, chatID int64)
}
```

例如，`Message` 类型是 Editable。以下是 Telebot 提供的 `StoredMessage` 类型的实现：
```go
// StoredMessage 是一个适合直接存储在数据库中或嵌入到较大结构体中的示例结构体，这通常是情况（您可能希望存储一些元数据，或者不需要）。
type StoredMessage struct {
	MessageID int   `sql:"message_id" json:"message_id"`
	ChatID    int64 `sql:"chat_id" json:"chat_id"`
}

func (x StoredMessage) MessageSig() (int, int64) {
	return x.MessageID, x.ChatID
}
```

为什么要这样做呢？嗯，它使您能够执行如下操作：
```go
// 数据库中只有两个整数列
var msgs []tele.StoredMessage
db.Find(&msgs) // gorm 语法

for _, msg := range msgs {
	bot.Edit(&msg, "Updated text")
	// 或者
	bot.Delete(&msg)
}
```

我认为这非常简洁。值得注意的是，此时还存在 Edit 系列中的另一个方法，`EditCaption()`，它使用得很少，因此我没有将其包含在 `Edit()` 中，就像我在 `SendAlbum()` 中所做的那样，因为这将不可避免地导致不必要的复杂性。

```go
var m *Message

// 更改照片、音频等的标题。
bot.EditCaption(m, "new caption")
```

## 键盘
Telebot 支持 Telegram 提供的回复键盘和内联键盘。任何按钮也可以作为 `Handle()` 的终点。

```go
var (
	// 通用标记构建器。
	menu     = &tele.ReplyMarkup{ResizeKeyboard: true}
	selector = &tele.ReplyMarkup{}

	// 回复按钮。
	btnHelp     = menu.Text("ℹ Help")
	btnSettings = menu.Text("⚙ Settings")

	// 内联按钮。
	//
	// 点击该按钮将导致客户端向机器人发送回调。
	//
	// 请确保 Unique 在按钮类型中保持唯一，因为它对于回调路由是必需的。
	//
	btnPrev = selector.Data("⬅", "prev", ...)
	btnNext = selector.Data("➡", "next", ...)
)

menu.Reply(
	menu.Row(btnHelp),
	menu.Row(btnSettings),
)
selector.Inline(
	selector.Row(btnPrev, btnNext),
)

b.Handle("/start", func(c tele.Context) error {
	return c.Send("Hello!", menu)
})

// 当按下Help按钮时（消息）
b.Handle(&btnHelp, func(c tele.Context) error {
	return c.Edit("Here is some help: ...")
})

// 当按下内联按钮时（回调）
b.Handle(&btnPrev, func(c tele.Context) error {
	return c.Respond()
})
```

您可以为每种可能的按钮类型使用标记构造器：
```go
r := b.NewMarkup()

// 回复按钮：
r.Text("Hello!")
r.Contact("Send phone number")
r.Location("Send location")
r.Poll(tele.PollQuiz)

// 内联按钮：
r.Data("Show help", "help") // 数据是可选的
r.Data("Delete item", "delete", item.ID)
r.URL("Visit", "https://google.com")
r.Query("Search", query)
r.QueryChat("Share", query)
r.Login("Login", &tele.Login{...})
```

## 内联模式
因此，如果要处理传入的内联查询，最好使用 `tele.OnQuery` 端点，然后使用 `Answer()` 方法将内联查询列表发送回去。在我编写此文档时，Telebot 支持提供的所有结果类型（但不支持缓存结果）。下面是示例：

```go
b.Handle(tele.OnQuery, func(c tele.Context) error {
	urls := []string{
		"http://photo.jpg",
		"http://photo2.jpg",
	}

	results := make(tele.Results, len(urls)) // []tele.Result
	for i, url := range urls {
		result := &tele.PhotoResult{
			URL:      url,
			ThumbURL: url, // 照片需要提供
		}

		results[i] = result
		// 需要为每个结果设置唯一的字符串ID
		results[i].SetResultID(strconv.Itoa(i))
	}

	return c.Answer(&tele.QueryResponse{
		Results:   results,
		CacheTime: 60, // 一分钟
	})
})
```

实际上没有太多可以讨论的。它还支持某种形式的身份验证，通过深度链接进行。为此，请使用 `QueryResponse` 的 `SwitchPMText` 和 `SwitchPMParameter` 字段。

# 贡献

1. Fork 该项目
2. 克隆 v3 分支: `git clone -b v3 https://github.com/tucnak/telebot`
3. 创建你的功能分支: `git checkout -b v3-feature`
4. 进行更改并添加它们: `git add .`
5. 提交更改: `git commit -m "add some feature"`
6. 推送到远程仓库: `git push origin v3-feature`
7. 创建 Pull Request

# 许可证

Telebot is distributed under MIT.