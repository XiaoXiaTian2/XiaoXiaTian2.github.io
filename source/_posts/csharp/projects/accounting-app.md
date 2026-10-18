---
title: 【C# 實作 06】AccountingApp：記帳本的 MVP、Repository 與 Builder
date: 2026-10-18 21:00:00
updated: 2026-10-18 21:00:00
cover: assets/csharp-cover.jpg
categories:
  - C#
  - 專案實作
tags:
  - C#
  - 設計模式
  - LINQ
  - 反射
  - WinForms
  - 專案實作
---

AccountingApp 是整個課程中規模最大的綜合專案：一個可以記帳、附上收據圖片、在表格中修改刪除，並做分組統計與圖表分析的 WinForms 記帳本。這篇整理它的架構，以及我在裡面用到的設計模式、LINQ 與反射。

<!-- more -->

# 專案簡介

功能分成四個畫面，由上方的 Navbar 切換：

| 畫面 | 功能 |
| --- | --- |
| 記一筆 | 輸入日期、金額、類別、細項、對象、付款方式，並上傳兩張圖片 |
| 記帳本 | 依日期區間查詢，在 DataGridView 中直接修改（ComboBox 欄位）、雙擊看大圖、刪除 |
| 帳戶分析 | 依類型／對象／付款方式分組、勾選篩選條件，統計金額 |
| 圖表分析 | 同樣的分組與篩選，產生折線圖、圓餅圖或堆疊長條圖 |

- 平台：**.NET Framework 4.8**、WinForms
- NuGet 套件：`AutoMapper` 10.0.0
- 參考：`System.Windows.Forms.DataVisualization`（圖表）、自己寫的 `CSV` 類別庫（`..\CSV\bin\Debug\CSV.dll`）
- 資料儲存：每天一個資料夾，裡面放 `data/data.csv` 與 `img/` 圖片

# 架構與功能

```text text
AccountingApp/
├── Views/            記一筆、記帳本、帳戶分析、圖表分析、ImageForm、SingletonForm
├── Components/       Navbar（UserControl）
├── Contracts/        各畫面的 IXxxPresenter / IXxxView，以及 IChartBuilder
├── Presenters/       AddAccountPresenter、ReordBookPresenter、AnalyzePresenter、ChartPresenter
├── Services/         AnalyzeService：篩選、分組、加總
├── Builders/         LineChartBuilder、PieChartBuilder、StackedColumnChartBuilder
├── Models/
│   ├── DAOs/         RecordModelDAO（對應 CSV 欄位）
│   ├── DTOs/         RecordModelDTO、AccountBookDTO、AnalyzeDTO、ComboxDataDTO
│   ├── Repositories/ IRecordRepository / RecordRepository、IDataRepository / DataRepository
│   └── RecordModel.cs、DataModel.cs
├── Attributes/       ComboBoxColumnAttribute、ImageColumnAttribute
├── Extensions/       DataGridViewExtension、FlowLayoutPanelExtension
└── Utility/          ImgCompressor、FormExtension（DebounceClick）
```

一次操作的資料流大致是：

```text text
View（Form）→ Presenter → Service / Builder → Repository → CSV 檔
          ←  OnXxx 回呼（DTO） ←
```

# 核心知識點

## 1. MVP 的 Contract 寫法

每個畫面都有一個 Contract，把 Presenter 與 View 的介面放在一起，Form 只實作 View 介面、只呼叫 Presenter 介面：

```csharp csharp
public class RecordBookContract
{
    public interface IRecordBookPresenter
    {
        void GetRecords(DateTime start, DateTime end);
        void EditRecords(AccountBookDTO accountBookDTO);
        void DeleteAccount(AccountBookDTO accountBookDTO);
    }

    public interface IRecordBookView
    {
        void OnGetRecords(List<AccountBookDTO> recordModelDTOs);
        void OnEditRecords(AccountBookDTO accountBookDTO);
        void OnDeleteAccount(List<AccountBookDTO> accountBookDTOS);
    }
}
```

Presenter 從 Repository 拿到 DAO 後，用 AutoMapper 套件轉成 DTO 再回呼 View。屬性名稱相同時不用逐一設定，註解裡留著我當時的筆記：

```csharp csharp
public void GetRecords(DateTime start, DateTime end)
{
    List<RecordModelDAO> daos = recordRepo.GetRecords(start, end);
    // ...
    var config = new MapperConfiguration(cfg =>
        cfg.CreateMap<RecordModelDAO, AccountBookDTO>()
        // 名稱相同可不寫
        //.ForMember(x => x.Date, y => y.MapFrom(o => o.Date))
        // ...
        );
    var mapper = config.CreateMapper();

    this._view.OnGetRecords(mapper.Map<List<AccountBookDTO>>(daos));
}
```

新增記錄時，DTO 裡的圖片是 `Image` 物件，但存進 CSV 的 DAO 只需要檔案路徑，這時就用 `ForMember` 指定轉換方式：

```csharp csharp
var config = new MapperConfiguration(cfg =>
    cfg.CreateMap<RecordModelDTO, RecordModelDAO>()
    .ForMember(x => x.Img1, y => y.MapFrom(o => path1))
    .ForMember(x => x.Img2, y => y.MapFrom(o => path2))
);
```

這也是後來我想自己實作一個 Mapper 的起點（下一篇）。

## 2. Repository：把「檔案怎麼存」藏起來

Presenter 和 Service 只依賴 `IRecordRepository`，不需要知道資料是一天一個資料夾的 CSV：

```csharp csharp
internal interface IRecordRepository
{
    bool CreateAccount(RecordModelDAO accountDAO);
    List<RecordModelDAO> GetRecords(DateTime start, DateTime end);
    RecordModelDAO EditRecords(RecordModelDAO recordDAO);
    List<RecordModelDAO> DeleteAccount(RecordModelDAO accountDAO);
}
```

查詢區間時，逐日組出資料夾路徑，有檔案才讀：

```csharp csharp
public List<RecordModelDAO> GetRecords(DateTime start, DateTime end)
{
    var diff = end.Date - start.Date;
    List<RecordModelDAO> records = new List<RecordModelDAO>();
    for (int i = 0; i <= diff.Days; i++)
    {
        string basePath = Path.Combine(baseFolder, start.AddDays(i).ToString("yyyy-MM-dd"));
        // ...
        string filePath = $"{basePath}/data/data.csv";
        if (File.Exists(filePath))
        {
            records.AddRange(CSVHelper.Read<RecordModelDAO>(filePath));
        }
    }
    return records;
}
```

修改記錄時，先讀出當天全部資料，用反射把新 DAO 的每個屬性複製到目標物件，再整份寫回：

```csharp csharp
RecordModelDAO target = records.FirstOrDefault(rec => rec.Img1 == recordDAO.Img1);
var props = target.GetType().GetProperties();
for (int i = 0; i < props.Length; i++)
{
    props[i].SetValue(target, props[i].GetValue(recordDAO));
}

CSVHelper.WriteList<RecordModelDAO>(filePath, records);
```

下拉選單的選項（類別、細項、對象、付款方式）也抽成 `IDataRepository`，之後如果要改成從資料庫讀取，只要換掉實作。

## 3. Builder 流暢介面 + 反射建立 Builder

三種圖表的建立步驟相同：設定時間、篩選、分組 → 取資料 → 分析 → 產生 Chart，差別只在資料形狀與畫法。所以我定義了一個回傳自己的 `IChartBuilder`：

```csharp csharp
public interface IChartBuilder
{
    IChartBuilder SetTimeRange(DateTime start, DateTime end);
    IChartBuilder SetFilters(Dictionary<string, List<string>> filters);
    IChartBuilder SetGroups(List<string> groups);

    IChartBuilder GetData();

    IChartBuilder AnalyzeData();

    Chart Build();
}
```

`ChartPresenter` 依照下拉選單選到的 `ChartType` 列舉值組出類別名稱，用 `Activator.CreateInstance` 建立對應的 Builder；折線圖需要「每週／每月」參數，所以呼叫帶參數的建構子：

```csharp csharp
public void GetChartDataDTO(DateTime start, DateTime end, Dictionary<string, List<string>> filters, List<string> groups, string period, ChartType chartType)
{
    Type classType = Type.GetType($"AccountingApp.Builders.{chartType}ChartBuilder");

    IChartBuilder builder = chartType == ChartType.Line
            ? (IChartBuilder)Activator.CreateInstance(classType, new object[] { period })
            : (IChartBuilder)Activator.CreateInstance(classType);

    Chart chart = builder.SetTimeRange(start, end).SetFilters(filters).SetGroups(groups).GetData().AnalyzeData().Build();

    _view.OnGetChartDataDTO(chart);
}
```

新增一種圖表只要加一個列舉值和一個 `XxxChartBuilder`，Presenter 不用改。各 Builder 也能有自己的細節，例如折線圖的 `GetData` 會多抓一段「前一期」的資料，畫成「本週／上週」兩條線比較。

## 4. 自訂 Attribute + 反射產生 DataGridView 欄位

記帳本的表格裡，有些欄位要用下拉選單、有些要顯示圖片。我沒有在設計工具裡逐欄設定，而是在 Model 上標註 Attribute：

```csharp csharp
internal class RecordModel
{
    [DisplayName("日期")]
    public string Date { get; set; }
    [DisplayName("錢錢")]
    public string Money { get; set; }
    [DisplayName("成員")]
    [ComboBoxColumn]
    public string Members { get; set; }
    // ...
    [DisplayName("圖片一")]
    [ImageColumn]
    public string Img1 { get; set; }
    // ...
}
```

畫面重繪時掃描屬性，依 Attribute 呼叫不同的擴充方法建立欄位：

```csharp csharp
foreach (var prop in typeof(RecordModel).GetProperties())
{
    // 擴充方法
    if (prop.GetCustomAttributes().Any((x) => x.GetType() == typeof(ComboBoxColumnAttribute)))
    {
        DataGridViewExtension.CreateComboxColumn(dataGridView1, prop);
        continue;
    }
    if (prop.GetCustomAttributes().Any((x) => x.GetType() == typeof(ImageColumnAttribute)))
    {
        DataGridViewExtension.CreateImageColumn(dataGridView1, prop);
        continue;
    }
    DataGridViewExtension.CreateTextBoxColumn(dataGridView1, prop);
}
```

ComboBox 欄位的選項也是用反射取得：屬性名稱 `Members` 剛好對應 `DataModel` 上同名的 static 欄位，標題則讀 `DisplayNameAttribute`：

```csharp csharp
if (prop.Name != "Details")
{
    comboBoxColumn.DataSource = typeof(DataModel).GetField(prop.Name).GetValue(null);
}
```

「靠命名慣例連結」讓新增欄位幾乎不用改 View，但代價是改名時編譯器幫不上忙。

## 5. LINQ 分析：OR / AND 混合篩選與動態分組

`AnalyzeService.AnalyzeData` 是我花最多時間想的地方。篩選規則是：**類別之間用 OR**（勾 Food 的「午餐」和 Move 的「悠遊卡」，兩種記錄都要出現），**其他欄位用 AND**（再勾「現金」就只留現金付款的）：

```csharp csharp
var categoriesKeys = DataModel.Details.Keys.ToList(); // 類別層級
// category 層級 filter 先合併成一組 OR
var categoryFilters = filters.Where(x => categoriesKeys.Contains(x.Key)).ToList();
if (categoryFilters.Count > 0)
{
    data = data.Where(x => categoryFilters.Any(filter =>
        filter.Key == x.Categories && filter.Value.Contains(x.Details)))
        .ToList();
}

// 其他層級用 AND
foreach (var filter in filters.Where(x => !categoriesKeys.Contains(x.Key)))
{
    PropertyInfo prop = typeof(RecordModelDAO).GetProperty(filter.Key);
    if (prop == null)
        continue;
    data = data.Where(x => filter.Value.Contains(prop.GetValue(x)?.ToString())).ToList();
}
```

分組則是依使用者勾選的維度，動態組出一個以逗號串接的 key，再 `GroupBy` 加總。多個類別都屬於「類型」這一軸，所以只加一次 `x.Details`：

```csharp csharp
.GroupBy(x =>
{
    var keyParts = new List<string>();
    bool categoryKeyAdded = false;
    foreach (var group in groups)
    {
        // 多個類別屬於同一個「類型」軸，只需算一次 x.Details，避免重複 key
        if (categoriesKeys.Contains(group))
        {
            if (categoryKeyAdded)
                continue;
            keyParts.Add(x.Details);
            categoryKeyAdded = true;
            continue;
        }

        keyParts.Add(x.GetType().GetProperty(group).GetValue(x, null).ToString());
    }
    return string.Join(",", keyParts);
})
.Select(x => new AnalyzeDTO(x.Key, x.Sum(y => int.Parse(y.Money))))
```

同樣的篩選邏輯在 `AnalyzeChartData`、`AnalyzeColumnStackData` 裡各複製了一份，之後應該抽成共用方法。

## 6. SingletonForm 與 Navbar：用反射做頁面切換

每個畫面只建立一次，存在 Dictionary 裡，切換時隱藏目前的 Form、顯示目標 Form。Form 依名稱用反射建立，並讀取 private 欄位 `navbar1`，把目前頁面的按鈕停用：

```csharp csharp
public static Form GetInstance(string formName)
{
    if (currentForm != null) { currentForm.Hide(); }

    if (!forms.TryGetValue(formName, out currentForm))
    {
        Type formType = Type.GetType($"AccountingApp.Views.{formName}");
        currentForm = (Form)Activator.CreateInstance(formType);
        currentForm.FormClosing += ClosingForm;

        Navbar nbr = (Navbar)currentForm.GetType().GetField("navbar1", BindingFlags.NonPublic | BindingFlags.Instance).GetValue(currentForm);
        nbr.DisableBtn(formName);

        forms.Add(formName, currentForm);
    }

    return currentForm;
}
```

Navbar 本身則用 `Assembly.GetExecutingAssembly().DefinedTypes` 掃描整個 Assembly，找出「繼承 Form 且含有 `navbar1` 欄位」的類別，自動產生按鈕。因為 Form 的類別名稱就是中文頁名，按鈕文字直接用 `type.Name`；新增頁面時只要在 Form 上拖一個 Navbar，選單就會自動出現。

# 小結

AccountingApp 是我第一次把 MVP、Repository、Builder、DTO/DAO 分層、AutoMapper、反射與 LINQ 全部放進同一個專案。分層後各畫面確實比較好擴充，例如新增圖表只要加一個 Builder。

---

- 系列目錄：{% post_link csharp/csharp-course-review 'C# 課程實作總整理' %}
- 上一篇：{% post_link csharp/desktop/student-management '【C# 實作 05】StudentManagement' %}
- 下一篇：{% post_link csharp/basics/automapper '【C# 實作 07】AutoMapper' %}
