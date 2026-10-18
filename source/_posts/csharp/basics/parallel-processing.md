---
title: 【C# 實作 08】ParallelProcessing：千萬筆 CSV 平行讀寫實驗
date: 2026-10-18 21:20:00
updated: 2026-10-18 21:20:00
cover: assets/csharp-cover.jpg
categories:
  - C#
  - 語言基礎
tags:
  - C#
  - 非同步
  - LINQ
---

系列的最後一篇是一個效能實驗：把最多 1,500 萬筆的 CSV 分批讀出再寫回，比較循序處理、平行處理、加不加 lock，以及不同批次大小的差異。程式不長，重點在實驗設計與數據。

<!-- more -->

# 專案簡介

- 平台：**.NET Framework 4.8**、Console
- 參考：自己寫的 `CSV` 類別庫（`..\CSV\bin\Debug\CSV.dll`），提供 `CSVHelper.Read<T>` 與 `CSVHelper.WriteList`
- 資料：`MOCK_DATA_{筆數}.csv`，欄位為 `id`、`first_name`、`last_name`、`email`、`gender`、`ip_address`
- 實驗環境（README 記載）：i7-8550U（4 核 8 執行緒）、32 GB RAM
- Repo：[https://github.com/XiaoXiaTian2/ParallelProcessing](https://github.com/XiaoXiaTian2/ParallelProcessing)

# 架構與功能

| 檔案 | 角色 |
| --- | --- |
| `Program.cs` | 設定筆數與批次大小，分批平行讀寫，統計時間 |
| `DataModel.cs` | CSV 一列對應的資料模型 |
| `ListExtensions.cs` | `Medium<T>` 擴充方法：計算中位數 |
| `README.md` | 各組實驗的數據記錄 |

程式流程：

```text text
計算批次數 → 清空輸出資料夾 → 每批一個 Task.Run
  每批：讀取第 N 段資料（計時）→ lock 後寫入輸出檔（計時）
→ Task.WhenAll 等全部完成 → 輸出讀取、寫入時間的中位數與總時間
```

repo 只有 3 個 commit，剛加入專案檔案時 `Main` 還是空的，最後一個 commit「分批上鎖寫入」才是現在的平行版本，所以 README 中「循序讀寫」那一組實驗所用的程式並不在 repo 裡，以下對程式碼的說明都以現在的平行版本為準。

# 核心知識點

## 1. 分批 + Task.Run + Task.WhenAll

一次把千萬筆讀進記憶體再處理，記憶體與時間都很可觀，所以先依 `batchSize` 切成多批，每批交給一個 Task 在執行緒集區上跑，`Main` 本身是 `async Task`，最後 `await Task.WhenAll` 等所有批次完成：

```csharp csharp
static async Task Main(string[] args)
{
    object a = new object();
    // ...
    int count = 15_000_000;
    int batchSize = 1_000_000;
    string inputPath = $"E:/csharp_course/data/MOCK_DATA_{count}.csv";
    string outputDir = "E:/csharp_course/outputData";

    // 計算分批
    int batchCount = (int)Math.Ceiling((double)count / batchSize);
    // ...
    List<Task> tasks = new List<Task>();
    for (int i = 0; i < batchCount; i++)
    {
        int batchNumber = i;

        // 每批處理
        Task task = Task.Run(() =>
        {
            // ...
        });

        tasks.Add(task);
    }

    await Task.WhenAll(tasks); // 等全部的批次都做完
    totalStopwatch.Stop();
    // ...
}
```

`Math.Ceiling((double)count / batchSize)` 先轉成 `double` 再無條件進位，筆數不能整除時最後一批才不會漏掉。數字分隔符號 `15_000_000` 則讓大數字一眼就能看出位數。

## 2. 迴圈變數的閉包陷阱

上面那行 `int batchNumber = i;` 看似多餘，其實是關鍵。Lambda 捕捉的是**變數**而不是當下的值；`for` 迴圈的 `i` 在整個迴圈中只有一個，等 Task 真正開始執行時，迴圈可能已經跑完，`i` 已經變成 `batchCount`，所有批次都會讀到同一段（甚至超出範圍）。在迴圈內宣告區域變數，每一圈都是新的變數，各個 Lambda 才會各自捕捉到正確的批次編號：

```csharp csharp
int batchNumber = i;

// 每批處理
Task task = Task.Run(() =>
{
    // ...
    Stopwatch stopwatch = Stopwatch.StartNew();
    List<DataModel> batch = CSVHelper.Read<DataModel>(inputPath, batchNumber * batchSize, batchSize);
    readTimes[batchNumber] = stopwatch.ElapsedMilliseconds / 1000.0;
    // ...
});
```

計時結果寫進 `readTimes[batchNumber]`：每個 Task 只寫自己那一格，不同 Task 不會寫到同一個位置，所以這裡不需要 lock。

## 3. lock 保護共用的寫入

讀取可以各讀各的，但所有批次都寫到同一個輸出檔，同時寫入會互相干擾，所以寫入的部分用 `lock` 包起來，一次只讓一個 Task 進去：

```csharp csharp
//string outputPath = $"{outputDir}/{count}/MOCK_DATA_{count}_{batchNumber + 1}.csv";
string outputPath = $"{outputDir}/{count}/MOCK_DATA_{count}.csv";
// ...
stopwatch.Restart();
lock (a)
{
    CSVHelper.WriteList(outputPath, batch);
}
writeTimes[batchNumber] = stopwatch.ElapsedMilliseconds / 1000.0;
```

被註解掉的那行是「每批寫成獨立檔案」的版本；那種寫法各批的輸出檔互不相干，本來就不需要 lock。README 的「無 lock」與「lock」兩組數據，推測就是在比較這兩種情境；不過 README 沒有寫明無 lock 版的寫檔方式，repo 裡也沒有那個版本的程式，無法確認。

## 4. 用 LINQ 寫中位數擴充方法

各批次的讀寫時間會有落差，我用中位數來代表「一批通常要多久」，好處是不會被少數特別慢的批次拉偏。寫成泛型擴充方法，透過 `selector` 決定要取哪個欄位：

```csharp csharp
public static double Medium<T>(this IEnumerable<T> list, Func<T, double> selector)
{
    var doubleList = list.Select(selector).OrderBy(x => x).ToList();

    int count = doubleList.Count();
    var middle = count / 2;
    if (count % 2 == 0)
    {
        double rent = doubleList.ElementAt(middle - 1) + doubleList.ElementAt(middle);
        return rent / 2;
    }

    return doubleList.ElementAt(middle);
}
```

```csharp csharp
double readTimeMedium = readTimes.Medium(x => x);
double writeTimeMedium = writeTimes.Medium(x => x);
double totalTime = totalStopwatch.ElapsedMilliseconds / 1000.0;
Console.WriteLine($"| 筆數 | 總讀取時間 | 總寫入時間 | 總時間 |\n| {count.ToString("N0")} | {readTimeMedium} | {writeTimeMedium} | {totalTime} |");
```

輸出直接是 Markdown 表格的一列，跑完就能貼進 README。

## 5. 實驗設計與數據

README 裡共有四組實驗：

1. **循序讀寫**：從 1 萬筆到 1,500 萬筆，記錄讀取、寫入、總時間與記憶體用量
2. **平行、無 lock**（批次 100 萬筆）：100 萬到 1,500 萬筆
3. **平行、有 lock**（批次 100 萬筆）：同樣的筆數區間
4. **實驗 2：批次大小**：批次從 100 萬到 700 萬筆，目標筆數 500 萬與 1,000 萬

另外還有一段「以 350 萬為一批次測試」的零散記錄，格式不完整，這裡就不引用。

以下節錄前三組的「總完成時間」欄位（單位：秒，數字照抄 README）。平行的兩組表格沒有表頭，這裡依循序表的欄位順序（筆數、讀取、寫入、總完成時間、記憶體）解讀：

| 筆數 | 循序 | 平行・無 lock | 平行・lock |
| --- | --- | --- | --- |
| 1,000,000 | 8.126 | 6.939 | 6.461 |
| 5,000,000 | 37.732 | 16.328 | 24.996 |
| 10,000,000 | 138.046 | 34.669 | 49.259 |
| 14,000,000 | 246.163 | 64.022 | 70.982 |
| 15,000,000 | — | 55.975 | 89.052 |

循序組的完整讀寫秒數與記憶體用量（節錄）：

| 筆數 | 讀取秒數(s) | 寫入讀取秒數(s) | 總完成時間(s) | 記憶體用量(mb) |
| --- | --- | --- | --- | --- |
| 1,000,000 | 5.984 | 2.142 | 8.126 | 282 |
| 5,000,000 | 26.255 | 11.477 | 37.732 | 1,300 |
| 10,000,000 | 70.308 | 67.738 | 138.046 | 2,600 |
| 14,000,000 | 150.787 | 95.376 | 246.163 | 3,700 |
| 15,000,000 | — | — | — | — |

實驗 2（批次大小）節錄目標 1,000 萬筆的部分：

| 批次筆數 | 目標筆數 | 讀取秒數(ms) | 寫入讀取秒數(ms) | 總完成時間(ms) | 記憶體用量(mb) |
| --- | --- | --- | --- | --- | --- |
| 1,000,000 | 10,000,000 | 11 | 14 | 119.219 | 2400 |
| 2,000,000 | 10,000,000 | 9.341 | 8.472 | 89.791 | 2200 |
| 3,500,000 | 10,000,000 | 14.976 /15.337 | 11.419 /11.283 | 77.893 /79.492 | 2400 /2400 |
| 5,000,000 | 10,000,000 | 61 | 63 | 124.226 | 2900 |
| 7,000,000 | 10,000,000 | 61 | 64 | 125.148 | 2700 |

幾點保守的觀察：

- 在節錄的筆數中，兩種平行版本的總時間都比循序版本短，而且筆數越多差距越明顯
- 除了 100 萬筆那一列之外，有 lock 的總時間都比無 lock 長；100 萬筆時只有一批，lock 不會有競爭，兩者接近也合理
- 循序版本在 1,500 萬筆那一列沒有數據；記憶體用量隨筆數增加，到 1,400 萬筆時記錄為 3,700 MB
- 實驗 2 目標 1,000 萬筆時，批次 350 萬筆的兩次測試（77.893 / 79.492）是總時間最短的；批次到 500 萬、700 萬時，讀寫時間跳到 60 以上，總時間也回到 120 以上

讀這些數據時有兩點要注意。第一，以目前的 `Program.cs` 來看，平行版本輸出的「讀取／寫入」是**每批時間的中位數**，不是全部批次的加總，所以只有「總完成時間」能和循序版本直接比較，上面的對照表也只放這一欄。第二，實驗 2 的表頭標示為 ms，但程式是用 `ElapsedMilliseconds / 1000.0` 換算後輸出；如果這組數據也是同一支程式跑出來的，數值實際上應該是秒。

# 待改進

1. **clone 後無法編譯與執行**：`ParallelProcessing.csproj` 參考的 `..\CSV\bin\Debug\CSV.dll` 不在 repo 裡；`Program.cs` 的輸入檔 `E:/csharp_course/data/MOCK_DATA_{count}.csv` 與輸出資料夾 `E:/csharp_course/outputData` 也寫死在本機。別人 clone 後既無法編譯，也沒有測試資料可跑。修正方向：把 CSV 專案納入 solution，路徑改由命令列參數傳入，並附上產生測試資料的方式。
2. **所有批次寫同一個檔案，輸出順序不固定**：每批獨立檔名的那行被註解掉，現在所有 Task 都寫入同一個 `MOCK_DATA_{count}.csv`。lock 只保證一次一批，不保證批次的先後；哪一批先讀完就先寫，所以輸出檔的資料列順序可能與輸入不同。另外 `CSVHelper` 的原始碼不在 repo，我無法從這裡確認 `WriteList` 對既有檔案是附加還是覆寫；若是覆寫，最後只會留下最後寫入的那一批。修正方向：恢復每批獨立檔名，或依批次編號依序寫入。
3. **寫入時間包含了等待 lock 的時間**：`stopwatch.Restart()` 放在 `lock (a)` 之前，所以 `writeTimes` 量到的是「排隊等鎖 + 實際寫入」。有 lock 版本的寫入時間偏高，有一部分可能只是排隊，並不代表寫入本身變慢。若要分開觀察，應該在取得鎖之後才開始計時，另外記錄等待時間。
4. **輸出標籤與實際計算不符**：`Console.WriteLine` 的表頭寫「總讀取時間」「總寫入時間」，實際印出的卻是 `readTimes.Medium(...)`、`writeTimes.Medium(...)` 的中位數。直接貼進 README 後，很容易誤把它當成加總，和循序版本的讀寫秒數放在一起比較。應該把表頭改成「讀取中位數」「寫入中位數」，或另外輸出加總。

# 小結

這個實驗讓我實際摸到 `async Task Main`、`Task.Run`、`Task.WhenAll`、閉包陷阱和 `lock`，也體會到「量測方式」本身就會影響結論：計時從哪裡開始、取中位數還是加總、單位標示是否正確，都會改變數據的意義。以後做效能比較，我會先把實驗程式、量測定義和資料產生方式一起放進 repo，讓結果可以被重現。

這也是 C# 實作系列的最後一篇，感謝看到這裡。

---

- 系列目錄：{% post_link csharp/csharp-course-review 'C# 課程實作總整理' %}
- 上一篇：{% post_link csharp/basics/automapper '【C# 實作 07】AutoMapper' %}
