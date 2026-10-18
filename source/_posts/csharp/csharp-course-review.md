---
title: C# 課程實作總整理：8 個 GitHub 專案的知識地圖
date: 2026-10-18 21:30:00
updated: 2026-10-18 21:30:00
cover: assets/csharp-cover.jpg
categories:
  - C#
tags:
  - C#
  - .NET
  - 學習筆記
  - 設計模式
  - 資料結構
---

這一年在 C# 課程中陸續完成了 8 個練習專案，這篇把它們一次整理起來，從程式碼回頭盤點學到了哪些知識點、還缺哪些，順便記錄重讀程式時發現的問題。

<!-- more -->

# 專案一覽

依最後 commit 日期排序，全部都是 **.NET Framework 4.8** 專案。

| 日期 | 專案 | 類型 | 主題 |
| --- | --- | --- | --- |
| 2025-08 | [OrderSystem](https://github.com/XiaoXiaTian2/OrderSystem) | WinForms | 點餐系統，多種折扣策略，並串接 Gemini AI 推薦最划算的折扣 |
| 2025-10 | [LinkedList](https://github.com/XiaoXiaTian2/LinkedList) | Console | 自己實作 LinkedList、二元搜尋樹、Dictionary 與迭代器 |
| 2025-11 | [Rx](https://github.com/XiaoXiaTian2/Rx) | Console | 觀察者模式、自製簡易版 Rx（Subject / Subscription） |
| 2025-12 | [YoutuberSubcribe](https://github.com/XiaoXiaTian2/YoutuberSubcribe) | WinForms | 把自製 Rx 套用到「頻道訂閱與通知」情境 |
| 2026-05 | [StudentManagement](https://github.com/XiaoXiaTian2/StudentManagement) | WPF | 學生資料增刪改，MVVM 搭配 MVP |
| 2026-07 | [AccountingApp](https://github.com/XiaoXiaTian2/AccountingApp) | WinForms | 記帳本：記錄、圖片、圖表分析，規模最大的綜合專案 |
| 2026-09 | [AutoMapper](https://github.com/XiaoXiaTian2/AutoMapper) | Console | 用反射與 Expression Tree 自己寫一個 AutoMapper |
| 2026-10 | [ParallelProcessing](https://github.com/XiaoXiaTian2/ParallelProcessing) | Console | 千萬筆 CSV 平行讀寫與效能實驗 |

# 系列文目錄

每個專案都有一篇獨立的文章，介紹架構、核心知識點與待改進的地方：

1. {% post_link csharp/projects/order-system '【C# 實作 01】OrderSystem：Strategy 折扣引擎與 Gemini Function Calling' %}
2. {% post_link csharp/data-structure/linkedlist '【C# 實作 02】LinkedList：自製串列、二元搜尋樹、Dictionary 與迭代器' %}
3. {% post_link csharp/design-pattern/rx '【C# 實作 03】Rx：從觀察者模式到自製 Subject / Subscription' %}
4. {% post_link csharp/desktop/youtuber-subscribe '【C# 實作 04】YoutuberSubcribe：把自製 Rx 套用到 WinForms' %}
5. {% post_link csharp/desktop/student-management '【C# 實作 05】StudentManagement：WPF 的 MVVM 搭配 MVP' %}
6. {% post_link csharp/projects/accounting-app '【C# 實作 06】AccountingApp：記帳本的 MVP、Repository 與 Builder' %}
7. {% post_link csharp/basics/automapper '【C# 實作 07】AutoMapper：用反射與 Expression Tree 自己寫 Mapper' %}
8. {% post_link csharp/basics/parallel-processing '【C# 實作 08】ParallelProcessing：千萬筆 CSV 平行讀寫實驗' %}

# 學習脈絡

```text text
資料結構＋迭代器 → 委派／事件／觀察者 → 自製 Rx → Rx 套用到 WinForms
  → WPF 的 MVVM → 綜合專案（記帳本：MVP＋Repository＋Builder＋AutoMapper 套件）
  → 反射＋Expression Tree（自己寫 AutoMapper）→ 平行處理與效能實驗
```

整體走的是「**先自己造輪子理解原理，再使用現成套件**」的路線：先自己寫 LinkedList、Dictionary，再用內建集合；先在記帳本裡使用 AutoMapper 套件，之後再自己實作一個。

# 知識重點

## 1. 資料結構與迭代器

專案：LinkedList、AutoMapper

- **泛型鏈結串列** `JackList<T>`：Add、Remove、Insert、AddRange、索引子 `this[int]`，用快慢指標偵測環
- **二元搜尋樹**：遞迴插入與刪除（刪除時以右子樹最小值替換），DFS / BFS 走訪（BFS 用 `Queue`）
- **雜湊表** `JackDictionary<TKey, TValue>`：bucket 陣列、鏈結法處理碰撞、負載因子超過 0.72 時擴容兩倍
- **迭代器協定**：自己實作 `IEnumerable` / `IEnumerator`（`MoveNext`、`Current`、`Reset`）
- **集合初始化器的原理**：`new Students { "Leo", "Allen" }` 需要型別實作 `IEnumerable` 並提供 `Add`
- 快速排序，以 Tuple 交換兩個值

```csharp csharp
// 雜湊值取正數再取餘數，決定放進哪個 bucket
int hashCode = key.GetHashCode() & 0x7FFFFFFF;
int index = hashCode % buckets.Length;
```

## 2. 泛型

- 泛型類別與泛型方法、多個型別參數（`Mapper.MapDest<TSource, TDest>`）
- 限制條件：`where T : class, new()`、`where TObeserver : IObeserver, new()`

## 3. 委派、事件與觀察者模式

專案：Rx、YoutuberSubcribe

- `Action<T>` / `Func<T>`、`event EventHandler<T>`、`?.Invoke`
- **運算子多載**：讓 `messenger += action` 可以註冊訂閱者
- 經典觀察者：`ISubject` / `IObserver`，Add / Remove / Notify
- **自製 Rx**：Subject、Observer（OnNext / OnError / OnComplete）、Subscription（Unsubscribe）
- 用 static 的 `MessageHandler` 當全域事件匯流排

```csharp csharp
Subject<string> subject = new Subject<string>();
Subscription<string> subscription = subject.Subscribe(data => Console.WriteLine(data));
subject.Next("1");
subscription.Unsubcribe();
```

## 4. 反射與 Expression Tree

專案：AutoMapper、AccountingApp、OrderSystem

這是所有專案裡最進階、也最有特色的部分：

- `Type.GetType` + `Activator.CreateInstance`：依命名慣例動態建立 Converter、Strategy、Form
- `PropertyInfo.GetValue` / `SetValue`；以 `BindingFlags.NonPublic` 呼叫 private 方法、讀取 private 欄位
- **解析 `Expression<Func<>>`**：Member、Unary、Binary、MethodCall、New、MemberInit、ListInit，等於寫了一個小型的運算式求值器
- 自訂 Attribute 搭配 `GetCustomAttribute`，自動產生 DataGridView 欄位
- 掃描 Assembly 找出所有子類別，自動註冊成 AI tool 宣告

```csharp csharp
// 依目標型別分類，組出 Converter 類別名稱後動態建立
PropertyType destPropertyType = destType.GetPropertyType();
string converterClassName = $"AutoMapper.Converters.{destPropertyType}Converter";
AConverter converter = (AConverter)Activator.CreateInstance(Type.GetType(converterClassName));
```

## 5. LINQ 與擴充方法

- `Select`、`Where`、`GroupBy`、`SelectMany`（帶索引）、`ToDictionary`、`Sum`、`FirstOrDefault`
- 記帳分析：多條件篩選（類別之間用 OR、其他欄位用 AND）、動態組合分組 key
- 擴充方法：`Medium<T>`（中位數）、`GetDigitalCount`、`DebounceClick`、WinForms 控制項產生器

## 6. 設計模式與架構

| 模式 | 出現位置 |
| --- | --- |
| Strategy | OrderSystem 的 4 種折扣策略，從 JSON 依類別名稱動態載入 |
| 抽象基底類別、範本方法 | `AStrategy`、`AConverter`、`AExpression`、`ATool<T>` |
| Observer | Rx、YoutuberSubcribe |
| Builder（流暢介面） | AccountingApp 的 `IChartBuilder`，產生折線圖、圓餅圖、堆疊長條圖 |
| Singleton / Registry | `SingletonForm`：快取 Form 實例並切換畫面 |
| Repository | `IRecordRepository`、`IDataRepository` |
| MVP（Contract 寫法） | AccountingApp、StudentManagement 的 `IXxxView` / `IXxxPresenter` |
| MVVM | StudentManagement（WPF） |
| DTO / DAO 分層 | AccountingApp、AutoMapper、StudentManagement |

## 7. 非同步與平行處理

專案：ParallelProcessing、OrderSystem、AccountingApp

- `async Task Main`、`Task.Run`、`Task.WhenAll`
- `lock` 保護共用的寫檔動作，並實測加 lock 與不加 lock 的效能差異
- 迴圈變數的閉包陷阱
- 用 `Stopwatch` 做 benchmark、比較不同批次大小
- WinForms 的 async 事件處理、`System.Threading.Timer` 實作 debounce、`form.Invoke` 切回 UI 執行緒

```csharp csharp
for (int i = 0; i < batchCount; i++)
{
    int batchNumber = i; // 先複製一份，避免 lambda 捕捉到同一個 i
    tasks.Add(Task.Run(() => ProcessBatch(batchNumber)));
}
await Task.WhenAll(tasks);
```

## 8. UI 開發

- **WinForms**：動態產生控制項、FlowLayoutPanel、DataGridView 自訂欄位（ComboBox、Image）、`DataVisualization.Charting` 圖表、用 `Tag` 關聯控制項
- **WPF**：XAML 的 Style、ControlTemplate、Trigger、`BasedOn`；Binding 與 `RelativeSource AncestorType`；`IValueConverter`；`ICommand` / `RelayCommand<T>`；`INotifyPropertyChanged` 與 Fody 自動通知

## 9. 檔案、資料與外部整合

- File、Directory、Path、FileStream；CSV 讀寫
- 圖片縮放、JPEG 品質壓縮（`EncoderParameter`）
- Newtonsoft.Json 序列化與反序列化（菜單、折扣規則放在 JSON）
- **Gemini API 的 Function Calling**：用 HttpClient 呼叫、宣告 tool、解析 functionCall 回應並轉成要執行的折扣策略
- 用 Windows Credential Manager 保存 API Key，不把金鑰寫死在程式碼裡

## 10. 零散語法點

`[Flags]` 列舉與位元運算、索引子、Tuple 回傳值 `(string, string)`、Pattern Matching（`is int number`）、數字分隔符號 `15_000_000`、字串插值、`?.`、巢狀介面、static class

# 回顧：待修正的地方

重讀程式碼時發現幾個問題，記下來之後修：

| 專案 / 檔案 | 問題 |
| --- | --- |
| `LinkedList/JackList.cs` 的 `DetectCycle` | `slow`、`fast` 一開始都指向 Head，第一圈就判定有環，串列非空時 AddRange 一定會被還原 |
| `LinkedList/Dict/JackDictionary.cs` 的 `GetValue` | 只比對 hashCode 沒比對 `key.Equals`，碰撞時會回傳錯誤的值 |
| `LinkedList/BinaryTree/BFSIterator.cs` | 讀取 `Current` 時改寫了 `Tree.Root`，走訪一次樹就被破壞 |
| `StudentManagement/Presenters/StudentPresenter.cs` | 沒有指定 `Id`，刪除時永遠刪到第一筆；建立 Student 時 Height 傳了兩次 |
| `OrderSystem/AI/AIAgent.cs` | `requestModel` 是 static，每次請求都會累積前幾次的 prompt |
| ParallelProcessing、AccountingApp | 依賴不在 repo 裡的 `CSV.dll`，且路徑寫死在本機磁碟，別人 clone 後無法編譯 |

# 下一步

和業界主流的 C# 課程比較，目前還沒涵蓋：

- **現代 .NET**：升級到 .NET 8/10，練習 `record`、Nullable Reference Types
- **資料庫**：SQL 與 EF Core，取代目前的 CSV 儲存
- **Web 後端**：ASP.NET Core Web API 與 DI 容器
- **單元測試**：替自製資料結構補上測試，順便驗證上面那些 bug 的修正
- **文件**：替每個 repo 補上 README
