---
title: 【C# 實作 03】Rx：從觀察者模式到自製 Subject / Subscription
date: 2026-10-18 20:30:00
updated: 2026-10-18 20:30:00
cover: assets/csharp-cover.jpg
categories:
  - C#
  - 設計模式
tags:
  - C#
  - 設計模式
  - 委派
  - 泛型
---

這篇整理委派、事件與觀察者模式的練習。從最經典的「報社與訂戶」寫起，接著用 `Action<T>` 把它簡化，最後仿照 Reactive Extensions（Rx）的介面，自己做一個迷你版的 Subject 與 Subscription。

<!-- more -->

# 專案簡介

Rx repo 裡有兩個專案：

- **EventHandlerPractice**（根目錄）：Console 程式，練習運算子多載、經典的 `ISubject` / `IObeserver` 觀察者模式
- **Rx**（`Rx/` 子資料夾）：類別庫，自製的 `Subject<T>`、`Observer<T>`、`Subscription<T>`，下一篇的 YoutuberSubcribe 就是引用它

共同資訊：

- 技術：.NET Framework 4.8
- 主題：`Action<T>`、`event EventHandler<T>`、運算子多載、泛型限制條件、觀察者模式
- Repo：[https://github.com/XiaoXiaTian2/Rx](https://github.com/XiaoXiaTian2/Rx)

# 架構與功能

```text text
Rx/
├── Program.cs             # EventHandlerPractice 的進入點
├── Messenger.cs           # 用 List<Action<T>> 收集訂閱者，並多載 + 運算子
├── ISubject.cs            # 經典觀察者模式：主題介面
├── IObeserver.cs          # 經典觀察者模式：觀察者介面
├── NewspaperOffice.cs     # 報社（主題），也多載了 + 運算子
├── MilkSender.cs          # 送牛奶（主題）
├── Customer.cs            # 訂戶（觀察者）
└── Rx/                    # 自製 Rx 類別庫
    ├── IObserver.cs       # OnNext / OnError / OnComplete
    ├── Observer.cs        # 用三個委派實作 IObserver
    ├── Subject.cs         # 管理觀察者、廣播資料
    ├── Subscription.cs    # 代表一次訂閱，可以取消
    └── Program.cs         # 使用範例
```

# 核心知識點

## 經典觀察者模式：Subject 與 Observer 介面

觀察者模式的角色很單純：主題（Subject）維護一份觀察者清單，狀態改變時逐一通知。我先用介面把角色定義出來：

```csharp csharp
internal interface ISubject<T>
{
    IList<T> observers { get; set; }
    void AddObserver(T observer);
    void RemoveObserver(T observer);
    void Notify();
}
```

報社和送牛奶都實作這個介面，`Notify` 時對每個訂戶呼叫 `Update`：

```csharp csharp
internal class MilkSender<TObeserver> : ISubject<TObeserver> where TObeserver : IObeserver, new()
{
    // ...
    public void Notify()
    {
        foreach (TObeserver obs in _observers)
        {
            obs.Update("牛奶");
        }
    }
    // ...
}
```

泛型限制 `where TObeserver : IObeserver, new()` 保證型別參數一定有 `Update` 方法可以呼叫，否則在泛型類別裡無法對 `TObeserver` 呼叫任何介面方法。好處是主題完全不需要知道訂戶的具體類別，`Program.cs` 裡同一個 `customerB` 可以同時訂報紙和牛奶。

## 運算子多載：讓訂閱寫成 +=

C# 的事件可以用 `+=` 註冊，我想讓自己的類別也能這樣寫，於是多載了 `+` 運算子（多載 `+` 之後，`+=` 會自動跟著可用）：

```csharp csharp
public static NewspaperOffice<TObeserver> operator +(NewspaperOffice<TObeserver> operand, TObeserver obeserver)
{
    operand.observers.Add(obeserver);
    return operand;
}
```

```csharp csharp
newspaperOffice += customerA;
newspaperOffice.AddObserver(customerB);
newspaperOffice.Notify();
```

`newspaperOffice += customerA` 會展開成 `newspaperOffice = newspaperOffice + customerA`，所以運算子一定要把 `operand` 回傳回去，否則變數會被換成別的東西。

## 用委派取代觀察者介面

經典寫法每一種訂戶都要寫一個類別。`Messenger<T>` 改成直接收集 `Action<T>`，訂閱者只要給一個 lambda：

```csharp csharp
public class Messenger<T>
{
    public event EventHandler<T> On;
    public List<Action<T>> events = new List<Action<T>>();

    public static Messenger<T> operator +(Messenger<T> operand, Action<T> action)
    {
        operand.events.Add(action);
        return operand;
    }

    public void Send(T msg)
    {
        Console.WriteLine($"origin msg: {msg}");

        foreach (var evt in events)
        {
            evt?.Invoke(msg);
        }
    }
}
```

這其實就是 `event` 在做的事：事件底層是一個多播委派，`+=` 就是把方法加進呼叫清單。手動寫一次之後，再看 `event EventHandler<T>` 就不再是黑盒子。

## 自製 Rx：Observer 用三個委派表達三種訊號

Rx 的觀察者會收到三種訊號：下一筆資料（OnNext）、錯誤（OnError）、結束（OnComplete）。我先定義介面，再用三個委派欄位實作，讓呼叫端可以只傳 lambda：

```csharp csharp
public interface IObserver<T>
{
    void OnNext(T num);
    void OnError(string e);
    void OnComplete();
}
```

```csharp csharp
public class Observer<T> : IObserver<T>
{
    public Action<T> Next;
    public Action<string> Error;
    public Action Complete;
    public void OnNext(T num)
    {
        Next?.Invoke(num);
    }
    public void OnError(string e)
    {
        Error?.Invoke(e);
    }
    public void OnComplete()
    {
        Complete?.Invoke();
    }
}
```

`?.Invoke` 讓沒有提供的 callback 安全地略過，這也是 `Subscribe` 可以把 `onError`、`onCompleted` 設成選擇性參數的原因。

## Subject 與 Subscription：訂閱的回傳值就是取消的鑰匙

`Subject<T>` 的 `Subscribe` 把委派包成 `Observer<T>` 加進清單，並回傳一個 `Subscription<T>`；`Next` 則把資料廣播給每個觀察者，任一個觀察者丟出例外時改呼叫它自己的 `OnError`，不會中斷其他人：

```csharp csharp
public Subscription<T> Subscribe(Action<T> onNext, Action<string> onError = null, Action onCompleted = null)
{
    IObserver<T> observer = new Observer<T>
    {
        Next = onNext,
        Error = onError,
        Complete = onCompleted,
    };
    _observers.Add(observer);

    return new Subscription<T>(this, observer);
}

public void Next(T num)
{
    foreach (IObserver<T> observer in _observers)
    {
        try
        {
            observer.OnNext(num);
        }
        catch (Exception e)
        {
            observer.OnError(e.Message);
        }
    }
}
```

`Subscription<T>` 只記住「哪個 Subject」和「哪個 Observer」，取消時請 Subject 移除它。`Subject.Remove` 宣告成 `internal`，引用這個類別庫的其他專案只能透過 Subscription 取消，不能任意刪別人的訂閱：

```csharp csharp
public class Subscription<T>
{
    private Subject<T> subject;
    private IObserver<T> _observer;
    public Subscription(Subject<T> subject, IObserver<T> observer)
    {
        this.subject = subject;
        _observer = observer;
    }

    public void Unsubcribe()
    {
        this.subject.Remove(_observer);
    }
}
```

使用起來就和 Rx 的 `IDisposable` 訂閱很像：

```csharp csharp
Subject<string> subject = new Subject<string>();

Subscription<string> subscriptionA = subject.Subscribe(data => Console.WriteLine(data));
Subscription<string> subscriptionB = subject.Subscribe(data => Console.WriteLine(data),
    err => Console.WriteLine(err), () => { });
subject.Next("1");
subscriptionA.Unsubcribe();
subject.Next("100");
subject.Complete();
subject.Next("1");
```

`"1"` 兩個訂閱者都會收到，`"100"` 只有 B 收到，`Complete()` 之後的 `"1"` 則沒有人收到。

# 待改進

1. **`Subject.Complete` 沒有通知觀察者**：`Complete()` 只有 `_observers.Clear();`，沒有先對每個 observer 呼叫 `OnComplete()`。結果是訂閱時傳入的 `onCompleted` 永遠不會被執行，訂閱者無從得知資料流已經結束。修正方向：先 `foreach` 呼叫 `OnComplete()` 再清空清單。
2. **在 `onNext` 裡取消訂閱會拋出例外**：`Subject.Next` 用 `foreach` 走訪 `_observers`，如果某個 `onNext` 裡呼叫了 `Unsubcribe()`（例如「收到第一筆就取消」），`Remove` 會修改正在走訪的 `List`，下一次迭代就拋出 `InvalidOperationException`。這個例外發生在 `foreach` 取下一個元素時，不在 `try` 範圍內，會直接中斷 `Next`。修正方向：走訪前先複製一份清單，例如 `foreach (var observer in _observers.ToList())`。
3. **沒有提供 `onError` 時例外會被默默吞掉**：`Next` 捕捉到例外後呼叫 `observer.OnError(e.Message)`，而 `Observer.OnError` 是 `Error?.Invoke(e)`。只傳 `onNext` 的訂閱者（例如範例中的 `subscriptionA`）出錯時什麼事都不會發生，除錯時很難發現。修正方向：沒有 `onError` 時重新拋出，或至少記錄下來。

# 小結

從 `ISubject` / `IObeserver` 到 `Action<T>`，再到 Subject / Subscription，同一個「通知」的需求寫了三次，每一次都把耦合再降低一點。自己實作過之後，比較能理解 Rx 為什麼要把訂閱設計成一個可以取消的物件，以及 OnComplete、OnError 這些訊號存在的意義，也因此更清楚目前這個迷你版還缺了哪些部分。

---

- 系列目錄：{% post_link csharp/csharp-course-review 'C# 課程實作總整理' %}
- 上一篇：{% post_link csharp/data-structure/linkedlist '【C# 實作 02】LinkedList' %}
- 下一篇：{% post_link csharp/desktop/youtuber-subscribe '【C# 實作 04】YoutuberSubcribe' %}
