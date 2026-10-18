---
title: 【C# 實作 01】OrderSystem：Strategy 折扣引擎與 Gemini Function Calling
date: 2026-10-18 20:10:00
updated: 2026-10-18 20:10:00
cover: assets/csharp-cover.jpg
categories:
  - C#
  - 專案實作
tags:
  - C#
  - 設計模式
  - 反射
  - WinForms
  - 專案實作
---

C# 實作系列的第一篇，從我在課程中第一個比較完整的 WinForms 專案開始：一個便當店點餐系統。它的重點不是畫面，而是「折扣規則怎麼寫才不會變成一大串 switch」，以及後來加上的 Gemini AI 自動挑選折扣。

<!-- more -->

# 專案簡介

OrderSystem 是一個點餐系統：左邊依菜單動態產生勾選框與數量欄位，右邊即時顯示訂單明細、總價，以及這次套用的折扣與理由。折扣可以手動從下拉選單選，也可以勾選「AI自動推薦」，把菜單和點餐內容丟給 Gemini，讓它挑出最划算的方案。

- 技術：.NET Framework 4.8、WinForms
- 套件：Newtonsoft.Json（菜單與折扣規則序列化）、Meziantou.Framework.Win32.CredentialManager（讀取 API Key）
- AI：Gemini API（`gemini-2.5-flash`）的 Function Calling

菜單與 22 種折扣規則都放在 `Menu.json`，路徑寫在 `App.config` 的 `Menu` 設定裡（目前是本機的絕對路徑，換電腦執行要先改成實際位置）。

# 架構與功能

```text text
OrderSystem/
├── Form1.cs                 # 主畫面：菜單勾選、折扣下拉選單、AI 推薦開關
├── Extension.cs             # 擴充方法：依菜單動態產生 FlowLayoutPanel 與控制項
├── Order.cs                 # 保存目前的訂單明細、計算總價
├── Discount.cs              # 清掉舊的折扣/贈品列，交給 DiscountContext 重新計算
├── DiscountContext.cs       # 依「手動」或「AI 推薦」決定要用哪個 Strategy
├── ShowPanel.cs             # 用反射把訂單明細轉成 Label
├── ReceivePanelHandler.cs   # static event，把結果送回 Form1
├── Strategies/              # 4 種折扣策略（AStrategy 子類別）
├── DiscountOrders/          # 早期版本：每種折扣一個類別
├── Models/                  # MenuModel、MenuData、OrderRequest、Item
└── AI/                      # Gemini 請求/回應模型、Tool 宣告與執行
```

| 策略類別 | 負責的折扣類型 | Menu.json 中的規則數 |
| --- | --- | --- |
| `BuyandFreeStrategy` | 買 N 送 M（可指定送最便宜、最貴或隨機） | 9 |
| `DiscountPriceStrategy` | 組合價、折價 | 5 |
| `DiscountPercentStrategy` | 指定品項組合打折 | 5 |
| `ConsumptionAmountStrategy` | 消費滿額打折、全場折扣 | 3 |

資料流大致是：使用者改數量 → `Order.AddOrder` 更新明細 → `Discount.DiscountOrder` 移除上次的折扣列 → `DiscountContext.HandleDiscount` 選出策略並把折扣/贈品列加回明細 → `ShowPanel` 產生畫面 → 透過 `ReceivePanelHandler.RenderPanel` 事件更新 Form1。

# 核心知識點

## Strategy 模式：把折扣規則從程式碼搬到 JSON

`DiscountOrders/` 資料夾保留了最早的寫法：每一種折扣是一個類別（大部分繼承 `Discount` 並覆寫 `CalDisount`），折扣名稱和計算數字直接寫死在程式裡。

```csharp csharp
class AllFivePercentOff : Discount
{
    public override void CalDisount(List<Item> items)
    {
        items.RemoveAll(x => x.name.Contains("贈送") || x.name.Contains("折扣"));

        int totalPrice = items.Sum(item => int.Parse(item.total));

        double substract = (totalPrice * 0.05);
        int sub = (int)substract;

        Item discount = new Item("(折扣)全場消費一律95折", sub.ToString(), "1", "-" + sub.ToString());
        items.Add(discount);
    }
}
```

這樣每多一個活動就要多一個類別。後來我把折扣歸納成 4 種「計算方式」，每種是一個 Strategy，具體的品項、數量、折扣比例則改成 JSON 資料：

```csharp csharp
public abstract class AStrategy
{
    protected List<Item> items { get; set; }
    protected MenuModel.Discount discount { get; set; }
    public AStrategy(List<Item> items, MenuModel.Discount discount)
    {
        this.items = items;
        this.discount = discount;
    }

    public abstract void DiscountOrder();
}
```

```json json
{
  "Name": "香魚飯搭配海帶湯打八折",
  "Strategy": "OrderSystem.Strategies.DiscountPercentStrategy",
  "Conditions": [
    { "Name": "香魚飯", "Number": 1, "Total_Price": 0 },
    { "Name": "海帶湯", "Number": 2, "Total_Price": 0 }
  ],
  "Rewards": [
    { "Name": "香魚飯搭配海帶湯", "Number": 1, "Free_type": "", "Discount_Price": 0, "Discount_Percentage": 0.8, "Set_Price": 0 }
  ]
}
```

（JSON 為了版面把物件壓成一行，內容與 `Menu.json` 相同。）

每筆折扣自己帶著 `Strategy` 欄位，寫的是完整類別名稱。`DiscountContext` 拿到後用反射建立對應的策略，所以新增活動只要改 JSON，不用動程式：

```csharp csharp
if (orderRequest.AIRecommand == false)
{
    Type discountType = Type.GetType(orderRequest.DiscountType.Strategy);
    AStrategy strategy = (AStrategy)Activator.CreateInstance(discountType, new object[] { orderRequest.OrderList, orderRequest.DiscountType });
    strategy.DiscountOrder();

    return ("abc", "");
}
```

`Activator.CreateInstance(Type, object[])` 會依參數挑選建構子，所以每個 Strategy 都必須有 `(List<Item>, MenuModel.Discount)` 這個簽章，這也是 `AStrategy` 把建構子固定下來的原因。

## 用 LINQ 比對折扣條件

條件的 `Name` 可以用 `|` 寫多個候選品項（例如任選幾種飲料）。策略裡先用帶索引的 `SelectMany` 把條件攤平，索引當作條件編號，再和訂單做比對、分組：

```csharp csharp
var conditionItems = discount.Conditions.SelectMany((condition, ID) =>
{
    return condition.Name.Split('|')
    .Select(name => new
    {
        Name = name,
        ConditionID = ID,
        TargetQty = condition.Number
    });
}).ToList();
// ...
var matchItemGroup = matchConditionItems
    .GroupBy(x => new { x.ConditionId, x.TargetQty })
    .Select(condition => new
    {
        ConditionID = condition.Key.ConditionId,
        Qty = condition.Sum(x => x.Number),
        Price = condition.Sum(x => x.Price),
        TargetQty = condition.Key.TargetQty,

    }).Where(x => x.Qty >= x.TargetQty).ToList();

if (matchItemGroup.Count != discount.Conditions.Length)
{
    return;
}

int leastFreeSets = matchItemGroup.Select(item => new
{
    fold = item.Qty / item.TargetQty
}
).Min(x => x.fold);
```

（摘自 `DiscountPercentStrategy.DiscountOrder`。）

邏輯是：每個條件分組後的數量都要達標（`Count` 必須等於條件數），而能套用幾組優惠取決於「最不夠的那個條件」，所以用 `Min` 算出 `leastFreeSets`。這段在 4 個策略裡結構幾乎相同，差別只在最後怎麼產生折扣列。

## Function Calling：用反射自動註冊 Tool 宣告

Gemini 的 Function Calling 需要在請求裡附上 `tools.functionDeclarations`，描述有哪些函式、參數長什麼樣子。我把宣告做成抽象類別 `Functiondeclaration`，請求模型在建構時掃描整個 Assembly，自動把所有子類別加進去：

```csharp csharp
public AIRequestModel()
{
    var allTypes = Assembly.GetExecutingAssembly().DefinedTypes;

    var declarationTypes = allTypes
        .Where(t => typeof(Functiondeclaration).IsAssignableFrom(t) && t.IsClass && !t.IsAbstract)
        .ToList();
    var declarationClass = declarationTypes
        .Select(type => (Functiondeclaration)Activator.CreateInstance(type))
        .ToList();

    this.tools = new List<Tool> { new Tool { functionDeclarations = declarationClass } };
}
```

目前只有一個工具「選擇折扣策略」，它的 `name` 刻意填成負責執行的類別完整名稱：

```csharp csharp
internal class DiscountSelectorDeclaration : Functiondeclaration
{
    public DiscountSelectorDeclaration() { }

    public override string name => "OrderSystem.AI.Tools.DiscountSelector.DiscountSelectorTool";

    public override string description => "選擇折扣策略";

    public override Parameters parameters
    {
        get => new DiscountSelectorParameters();
    }
}
```

參數（`name`、`strategy`、`reason`）定義在 `DiscountSelectorParameters`，三個都是 required。之後要加新工具，只要再寫一組 Declaration / Tool 類別，不用改請求模型。

## 解析 functionCall 並用反射執行

模型回應時，如果 `text` 是 null，代表它選擇呼叫函式，`ExecRequest` 就回傳帶有 `functionCall` 的結果：

```csharp csharp
if (aIResponse.candidates[0].content.parts[0].text == null)
{
    return new AIResult(true, null, aIResponse.candidates[0].content.parts[0].functionCall);
}
return new AIResult(false, aIResponse.candidates[0].content.parts[0].text, null);
```

因為宣告時 `name` 就是類別名稱，執行時直接 `Type.GetType(name)` 建立工具並呼叫 `Apply`：

```csharp csharp
public object UseTool()
{
    string name = functioncall.name;
    Console.WriteLine(JsonConvert.SerializeObject(functioncall));

    Type toolType = Type.GetType(name);
    object strategy = Activator.CreateInstance(toolType, new object[] { functioncall.args });

    MethodInfo applyFunc = strategy.GetType().GetMethod("Apply", BindingFlags.Public | BindingFlags.Instance);
    return applyFunc.Invoke(strategy, null);
}
```

`functioncall.args` 反序列化後是 `object`（實際是 JObject），型別不明確。我在泛型基底 `ATool<T>` 裡先序列化成字串、再反序列化成工具需要的參數型別，讓子類別直接拿到強型別的 `response`：

```csharp csharp
public abstract class ATool<T>
{
    protected T response = default;
    protected Args Args { get; set; }

    public ATool(object respnose)
    {
        string responseString = JsonConvert.SerializeObject(respnose);
        this.response = JsonConvert.DeserializeObject<T>(responseString);
    }

    public abstract object Apply();
}
```

最後 `DiscountContext` 拿 AI 回傳的折扣名稱到 `MenuData.Discounts` 找出規則，再用和手動模式一樣的方式建立 Strategy。AI 只負責「選」，實際金額仍由程式計算，避免模型算錯錢。

## API Key 放在 Windows 認證管理員

金鑰沒有寫進程式碼或 `App.config`（註解裡還留著原本讀 `AppSettings` 的寫法），而是從 Windows Credential Manager 讀取：

```csharp csharp
//private static readonly string _apiKey = ConfigurationManager.AppSettings["apiKey"];
private static readonly string _apiKey = CredentialManager.ReadCredential(applicationName: "GeminiAppKey").Password;
```

這樣程式碼分享出去也不會外洩金鑰；代價是執行前要先在認證管理員新增一筆名為 `GeminiAppKey` 的認證。

## WinForms：動態產生控制項並用 Tag 互相關聯

菜單品項不是在設計工具裡拉的，而是 `Extension.GetMenu` 依 JSON 動態產生。勾選框和數量欄位透過 `Tag` 互相指向，事件處理時就能從一個控制項找到另一個：

```csharp csharp
CheckBox checkBox = new CheckBox
{
    Text = item1.Name + '$' + item1.Price,
    Padding = new Padding(5, 0, 0, 0),
    Size = new Size(100, 20),
    Location = new Point(0, yi),
};
yi += 30;
NumericUpDown numericUpDown = new NumericUpDown
{
    Size = new Size(40, 0),
    Tag = checkBox
};
checkBox.Tag = numericUpDown;
checkBox.CheckedChanged += CheckChanged;

numericUpDown.ValueChanged += NumberChanged;
```

計算完的明細則由 `ShowPanel` 包成 `FullSidePanel`，透過 static event 送回 Form1，讓折扣計算的程式不需要直接持有 Form 的參考：

```csharp csharp
class ReceivePanelHandler
{
    // 08/16 HW 跟理由區打包成物件
    public static event EventHandler<FullSidePanel> RenderPanel;

    public static void GetFlowPanel(FullSidePanel fullSidePanel)
    {
        RenderPanel.Invoke(null, fullSidePanel);
    }
}
```

# 小結

這個專案讓我第一次體會到 Strategy 模式的價值：把「規則的資料」和「計算方式」拆開後，22 種活動只需要 4 個類別。Function Calling 的部分則把反射用到了極致，宣告靠掃描 Assembly 自動註冊、執行靠類別名稱動態建立。

---

- 系列目錄：{% post_link csharp/csharp-course-review 'C# 課程實作總整理' %}
- 下一篇：{% post_link csharp/data-structure/linkedlist '【C# 實作 02】LinkedList' %}
