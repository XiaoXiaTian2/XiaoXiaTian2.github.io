---
title: 【C# 實作 04】YoutuberSubcribe：把自製 Rx 套用到 WinForms
date: 2026-10-18 20:40:00
updated: 2026-10-18 20:40:00
cover: assets/csharp-cover.jpg
categories:
  - C#
  - 桌面應用
tags:
  - C#
  - WinForms
  - 委派
  - 設計模式
---

上一篇自己做了一個迷你版 Rx，這篇把它拿來用：寫一個「YouTuber 發片、訂閱者收到通知」的 WinForms 小程式，看看 Subject / Subscription 放進實際的 UI 情境時長什麼樣子。

<!-- more -->

# 專案簡介

YoutuberSubcribe 有三個視窗：

- **消息大廳**（`NotifyForm`）：主視窗，左右兩欄分別顯示 YouTuber 端和訂閱者端發生的事件，並提供「Yotuber入口」與「User入口」兩個按鈕
- **YouTuber 視窗**（`YoutuberForm`）：選擇頻道、輸入影片標題與描述後上傳
- **訂閱者視窗**（`UserForm`）：輸入名稱訂閱頻道，列出該頻道的訂閱者，每個人旁邊有「取消訂閱」按鈕

技術與相依：

- .NET Framework 4.8、WinForms
- 直接引用上一篇 Rx 專案編譯出的 `Rx.dll`，使用其中的 `Subject<T>` 與 `Subscription<T>`

# 架構與功能

| 類別 | 角色 |
| --- | --- |
| `Youtuber` | 擁有 4 個頻道（ch1～ch4），`Publish` 時交給指定頻道發布 |
| `Channel` | 包一個 `Subject<Video>`，發布影片就是 `Subject.Next` |
| `Video` | 影片資料：名稱、描述 |
| `User` | 訂閱者，保存自己的 `Subscription<Video>`，收到影片時發一則訊息到消息大廳 |
| `MessageHandler` | static 類別，包一個 `Subject<Message>`，當作全域的訊息匯流排 |
| `Message`、`SenderType` | 訊息內容與來源（Youtuber / User） |
| `NotifyForm`、`YoutuberForm`、`UserForm` | 三個視窗 |

這裡其實有**兩層**觀察者關係：

```text text
YoutuberForm ──上傳──> Channel.Subject<Video> ──> User.Receive
     │                                              │
     └──────────> MessageHandler.Subject<Message> <─┘
                              │
                              └──> NotifyForm（顯示在消息大廳）
```

第一層是「頻道 → 訂閱者」，是這個題目本身的業務邏輯；第二層是「任何地方 → 消息大廳」，用來讓 UI 顯示發生了什麼事。

# 核心知識點

## 用 Subject 實作頻道

`Channel` 不自己管理訂閱者清單，直接把工作交給 Rx 的 `Subject<Video>`。發布影片就是 `Next`，訂閱就是把 `Subject.Subscribe` 的結果回傳出去：

```csharp csharp
public class Channel
{
    public readonly Subject<Video> Subject = new Subject<Video>();
    public string ChannelName { get; set; }
    public Channel(string name) { ChannelName = name; }
    public void Publish(Video video)
    {
        Subject.Next(video);
    }

    public Subscription<Video> Subscribe(Action<Video> onNext)
    {
        return Subject.Subscribe(onNext);
    }
}
```

和上一篇的 `NewspaperOffice` 比起來，這裡完全不用寫 `AddObserver`、`RemoveObserver`、`Notify`，觀察者模式的樣板程式都被 Subject 吸收掉了。`Channel.Subscribe` 只開放 `onNext`，因為這個情境用不到錯誤與結束訊號。

## 訂閱者保存自己的 Subscription

訂閱時把 `user.Receive` 這個**方法群組**直接當成 `Action<Video>` 傳進去，C# 會自動把它轉成委派。回傳的 `Subscription` 存在 `User` 身上，之後取消訂閱就靠它：

```csharp csharp
User user = new User(subscribername.Text, channel.ChannelName);
var subscription = channel.Subscribe(user.Receive);
user.Subscription = subscription;
```

```csharp csharp
public class User
{
    public string Name { get; set; }
    public string ChannelName { get; set; }
    public Subscription<Video> Subscription { get; set; }
    public User(string name, string channelName)
    { Name = name; ChannelName = channelName; }
    public void Receive(Video video)
    {
        if (video != null)
        {
            MessageHandler.Publish(new Message(Constants.SenderType.User, $"{Name} 收到來自 {ChannelName} 的新發布影片: {video._name}"));
        }
    }
}
```

`Channel` 不知道 `User` 的存在，它只認得一個 `Action<Video>`；這就是委派帶來的解耦：任何簽章相符的方法都能訂閱。

## static MessageHandler：全域訊息匯流排

三個視窗都需要把事件顯示到消息大廳，但我不想讓 `YoutuberForm`、`UserForm`、`User` 都拿到 `NotifyForm` 的參考。解法是一個 static 的 Subject，誰都可以 `Publish`，消息大廳只要訂閱一次：

```csharp csharp
internal static class MessageHandler
{
    public static readonly Subject<Message> Subject = new Subject<Message>();
    public static void Publish(Message message)
    {
        Subject.Next(message);
    }
}
```

`NotifyForm` 在建構子裡訂閱，依訊息來源決定加到哪一欄：

```csharp csharp
MessageHandler.Subject.Subscribe(message =>
{
    switch (message.Type)
    {
        case SenderType.Youtuber:
            youtubepanel.Controls.Add(new Label
            {
                Text = message.Data,
                Width = youtubepanel.Width
            });
            break;
        case SenderType.User:
            userpanel.Controls.Add(new Label
            {
                Text = message.Data,
                Width = userpanel.Width
            });
            break;
    }
});
```

這和 OrderSystem 用 static event `ReceivePanelHandler.RenderPanel` 把結果送回 Form1 是同一個想法，只是這次換成自己寫的 Subject。所有發布都發生在按鈕的 Click 事件裡，也就是 UI 執行緒上，所以 lambda 裡可以直接操作控制項，不需要 `Invoke`。

## 多個視窗共用同一份資料

`NotifyForm` 持有唯一的 `Youtuber` 實例，開啟其他視窗時把它（或它的頻道清單）從建構子傳進去，兩個視窗看到的就是同一批 `Channel` 物件：

```csharp csharp
private void button1_Click(object sender, EventArgs e)
{
    YoutuberForm youtuberForm = new YoutuberForm(Youtuber);
    youtuberForm.Show();
}

private void button2_Click(object sender, EventArgs e)
{
    UserForm userForm = new UserForm(Youtuber.channel);
    userForm.Show();
}
```

下拉選單直接綁定 `List<Channel>`，用 `DisplayMember` 指定顯示的屬性；因為沒有設定 `ValueMember`，`SelectedValue` 拿到的就是 `Channel` 物件本身，上傳時直接轉型使用：

```csharp csharp
private void upload_Click(object sender, EventArgs e)
{
    string fullText = $"{channelbox.Text} 發布 {videotitle.Text}";
    MessageHandler.Publish(new Message(SenderType.Youtuber, fullText));
    _youtuber.Publish(new Video(videotitle.Text, description.Text), (Channel)channelbox.SelectedValue);
}
```

## 動態產生訂閱者清單，用 Tag 綁定資料

`UserForm` 用 `ToDictionary` 替每個頻道建立一份訂閱者清單，切換頻道時重新繪製。每一列是動態產生的 Label 加按鈕，按鈕的 `Tag` 放著對應的 `User`，點擊時就知道要取消誰：

```csharp csharp
private void renderChannelList(List<User> users)
{
    flowLayoutPanel1.Controls.Clear();
    foreach (var user in users)
    {
        FlowLayoutPanel flowLayoutPanel = new FlowLayoutPanel() { Height = 25, Width = flowLayoutPanel1.Width, };
        flowLayoutPanel.Controls.Add(new Label() { Text = user.Name, BorderStyle = BorderStyle.Fixed3D, TextAlign = ContentAlignment.MiddleCenter, Width = 123, Height = 25 });
        Button btn = new Button() { Text = "取消訂閱", Width = 75, Tag = user };
        btn.Click += unsubscribe_Click;
        flowLayoutPanel.Controls.Add(btn);
        flowLayoutPanel1.Controls.Add(flowLayoutPanel);
    }
}
```

```csharp csharp
private void unsubscribe_Click(object sender, EventArgs e)
{
    Button btn = sender as Button;
    User user = btn.Tag as User;
    user.Subscription.Unsubcribe();
    flowLayoutPanel1.Controls.Remove(btn.Parent);
    string fullText = $"{user.Name} 取消訂閱";
    MessageHandler.Publish(new Message(SenderType.User, fullText));
}
```

所有按鈕共用同一個 `unsubscribe_Click`，靠 `sender` 和 `Tag` 區分是哪一列，不需要為每個訂閱者各寫一個 lambda。

# 小結

這個專案讓自製的 Rx 第一次有了「真正的使用者」：頻道訂閱用 Subject 表達業務關係，static 的 MessageHandler 則把多個視窗之間的通知解耦。

---

- 系列目錄：{% post_link csharp/csharp-course-review 'C# 課程實作總整理' %}
- 上一篇：{% post_link csharp/design-pattern/rx '【C# 實作 03】Rx' %}
- 下一篇：{% post_link csharp/desktop/student-management '【C# 實作 05】StudentManagement' %}
