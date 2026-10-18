---
title: 【C# 實作 07】AutoMapper：用反射與 Expression Tree 自己寫 Mapper
date: 2026-10-18 21:10:00
updated: 2026-10-18 21:10:00
cover: assets/csharp-cover.jpg
categories:
  - C#
  - 語言基礎
tags:
  - C#
  - 反射
  - Expression Tree
  - 泛型
---

在記帳本專案用過 AutoMapper 套件之後，我很好奇 `ForMember(x => ..., y => ...)` 這種寫法背後是怎麼運作的，於是自己動手寫了一個簡化版。這篇整理裡面用到的反射、Expression Tree 與泛型。

<!-- more -->

# 專案簡介

這是一個 Console 專案，目標是做到：

- 依屬性名稱自動把 `UserDTO` 轉成 `UserDAO`，型別不同時自動轉換（`float` → `int`、`int` → 列舉、`List<SchoolDTO>` → `List<SchoolDAO>`……）
- 支援 `ForMember`：用 Lambda 指定某個目標屬性要怎麼算
- 順便用同樣的技巧寫一個 `OrderBy2`，以 Lambda 指定排序欄位的快速排序

技術面：

- 平台：**.NET Framework 4.8**、Console
- 沒有使用任何 NuGet 套件，只用 `System.Reflection` 與 `System.Linq.Expressions`
- Repo：[https://github.com/XiaoXiaTian2/AutoMapper](https://github.com/XiaoXiaTian2/AutoMapper)

# 架構與功能

```text text
AutoMapper/
├── Mapper.cs              進入點 MapDest<TSource, TDest>，分派 Converter 與 Expressioner
├── MapperConfig.cs        ForMember 設定，記錄「目標屬性 → 來源運算式」
├── Converters/            依目標型別分類的轉換器
│   ├── AConverter.cs        抽象基底
│   └── Basic / Enum / Array / Enumerable / Class Converter
├── Expressions/           Expression Tree 求值器，每種節點一個類別
│   ├── AExpression.cs       抽象基底
│   └── Member / Constant / Parameter / Unary / Binary / MethodCall
│       / New / MemberInit / ListInit Expressioner
├── Extensions/            GetPropertyType、GetProperty、OrderBy2
├── Sorters/QuickSorter.cs 以 Expression 指定欄位的快速排序
└── Models/                UserDTO / UserDAO / SchoolDTO / SchoolDAO 測試資料
```

使用方式（`Program.cs`）：

```csharp csharp
List<UserDAO> users = Mapper.MapDest<UserDTO, UserDAO>(lists, config =>
{
    config
    // ...
    .ForMember(x => new List<SchoolDAO>() { new SchoolDAO() { Id = x.Id } }, y => y.schools);
}).ToList();
```

# 核心知識點

## 1. 依型別分類，用命名慣例動態建立 Converter

整個 Mapper 的核心是 `Convert(object source, Type destType)`：先判斷目標型別屬於哪一類，再用類別名稱字串建立對應的 Converter。

```csharp csharp
private static object Convert(object source, Type destType)
{
    if (source == null)
        return source;

    PropertyType destPropertyType = destType.GetPropertyType();
    string converterClassName = $"AutoMapper.Converters.{destPropertyType.ToString()}Converter";
    AConverter converter = (AConverter)Activator.CreateInstance(Type.GetType(converterClassName));

    return converter.Convert(source, destType);
}
```

分類規則寫成 `Type` 的擴充方法。注意判斷順序：`string` 也實作了 `IEnumerable<char>`，所以必須先判斷基本型別，否則字串會被當成集合處理。

```csharp csharp
if (basicTypes.Contains(type))
{
    return PropertyType.Basic;
}
else if (type.IsEnum)
{
    return PropertyType.Enum;
}
else if (type.IsArray)
{
    return PropertyType.Array;
}
else if (
    type.GetInterfaces()
    .Any(x => x.IsGenericType && x.GetGenericTypeDefinition() == typeof(IEnumerable<>))
   )
{
    return PropertyType.Enumerable;
}

return PropertyType.Class;
```

`PropertyType` 列舉的值（`Basic`、`Enum`、`Array`、`Enumerable`、`Class`）剛好對應 Converter 的類別名稱前綴，新增一種轉換只要加列舉值和類別，不用改 `Convert` 本身，這是類似 Strategy 的作法。

## 2. 遞迴轉換：複合型別交回 Mapper 處理每個元素

`ClassConverter` 建立目標物件後，逐一取出來源的同名屬性，再把值丟回 `Mapper.Convert` 轉成目標屬性的型別：

```csharp csharp
public override object Convert(object source, Type destType)
{
    PropertyInfo[] propertyInfo = destType.GetProperties();

    object destObject = Activator.CreateInstance(destType);
    foreach (var property in propertyInfo)
    {
        PropertyInfo sourceProp = source.GetType().GetProperty(property.Name);
        property.SetValue(destObject, MapValue(sourceProp.GetValue(source), property.PropertyType));
    }

    return destObject;
}
```

集合也是同樣的思路，`EnumerableConverter` 取出泛型參數作為元素型別，每個元素各自遞迴轉換：

```csharp csharp
public override object Convert(object source, Type destType)
{
    Type destElementType = destType.GetGenericArguments()[0];
    IList destList = (IList)Activator.CreateInstance(destType);

    foreach (object sourceElement in (IEnumerable)source)
    {
        destList.Add(MapValue(sourceElement, destElementType));
    }

    return destList;
}
```

`Mapper.Convert` 是 private，Converter 在另一個類別裡要怎麼呼叫？我在抽象基底 `AConverter` 用反射加上 `BindingFlags.NonPublic` 取得它。這樣 Mapper 對外只公開 `MapDest`，內部的遞迴入口不會被外部誤用：

```csharp csharp
public abstract class AConverter
{
    public abstract object Convert(object soruce, Type destType);

    protected static object MapValue(params object[] parameters)
    {
        MethodInfo method = typeof(Mapper).GetMethod("Convert", BindingFlags.NonPublic | BindingFlags.Static);
        return method.Invoke(null, parameters);
    }
}
```

## 3. 基本型別：用反射找 System.Convert.ToXxx

基本型別之間的轉換，我不想寫一大串 `if/switch`，而是利用 `System.Convert` 的命名規則：目標是 `Int32` 就找 `ToInt32`，並且用來源型別當參數型別找出正確的多載：

```csharp csharp
public override object Convert(object source, Type destType)
{
    if (destType.IsInstanceOfType(source))
    {
        return source;
    }

    if (source is Enum)
    {
        source = (int)source;
    }

    if (destType == typeof(string)) // string 沒有 Parse
    {
        return source.ToString();
    }

    MethodInfo parseMethod = typeof(System.Convert).GetMethod($"To{destType.Name}", new Type[] { source.GetType() }); if (parseMethod == null)
        throw new InvalidOperationException(destType.Name + " 沒有 Convert 方法");

    return parseMethod.Invoke(null, new object[] { source });
}
```

列舉先轉成 `int` 再交給 `Convert`；反方向（`int` → 列舉）則由 `EnumConverter` 用 `Enum.Parse(destType, source.ToString())` 處理，所以 `UserDTO.Gender`（`int`）可以轉成 `UserDAO.Gender`（`Gender` 列舉）。

## 4. ForMember：用 Expression 取得「目標是哪個屬性」

`ForMember` 的兩個參數都是 `Expression<Func<...>>` 而不是 `Func<...>`。差別在於 `Func` 只能執行，`Expression` 則是一棵可以被程式讀取的語法樹。目標端 `y => y.schools` 的 Body 是 `MemberExpression`，從中可以拿到 `PropertyInfo`；來源端則先把整棵 Body 存起來，等到轉換時再求值：

```csharp csharp
public class MapperConfig<TSource, TDest>
{
    public Dictionary<PropertyInfo, ExpressionModel> memberLists = new Dictionary<PropertyInfo, ExpressionModel>();

    public MapperConfig<TSource, TDest> ForMember<TProp, SProp>(Expression<Func<TSource, SProp>> source, Expression<Func<TDest, TProp>> dest)
    {
        PropertyInfo destProperty = (PropertyInfo)((MemberExpression)dest.Body).Member;
        memberLists.Add(destProperty, new ExpressionModel() { Expression = source.Body });

        return this;
    }
}
```

`MapDest` 先依名稱做完整的自動轉換，再套用 `ForMember` 的設定覆蓋指定屬性：

```csharp csharp
public static TDest MapDest<TSource, TDest>(TSource source, Action<MapperConfig<TSource, TDest>> action = null)
{
    MapperConfig<TSource, TDest> config = new MapperConfig<TSource, TDest>();
    action?.Invoke(config); // add dictionary

    TDest dest = (TDest)Convert(source, typeof(TDest));
    ApplyMembers(config, source, dest);

    return dest;
}

// ...（IEnumerable 版的 MapDest 多載）

private static void ApplyMembers<TSource, TDest>(MapperConfig<TSource, TDest> config, TSource source, TDest dest)
{
    foreach (var member in config.memberLists)
    {
        object value = Express(member.Value.Expression, source);
        member.Key.SetValue(dest, Convert(value, member.Key.PropertyType));
    }
}
```

## 5. 自己寫一個 Expression Tree 求值器

`Express` 依節點型別分派到對應的 Expressioner。Unary、Binary 節點的 `NodeType` 有很多種（`Not`、`Convert`、`Add`、`Equal`……），Member、MethodCall 的 `NodeType`（`MemberAccess`、`Call`）又和 Expressioner 的類別名稱對不上，所以這四類用 `is` 判斷類別，其他節點則直接用 `NodeType` 組類別名稱：

```csharp csharp
private static object Express(Expression expression, object source)
{
    string expressionName = expression.NodeType.ToString();
    string expressionerClassName = null;
    if (expression is MemberExpression a)
    {
        expressionerClassName = "AutoMapper.Expressions.MemberExpressioner";
    }
    else if (expression is UnaryExpression)
    {
        expressionerClassName = "AutoMapper.Expressions.UnaryExpressioner";
    }
    // ...
    else { expressionerClassName = $"AutoMapper.Expressions.{expressionName}Expressioner"; }
    AExpression converter = (AExpression)Activator.CreateInstance(Type.GetType(expressionerClassName));

    return converter.Evaluate(expression, source);
}
```

每個 Expressioner 只處理自己那種節點，子節點再遞迴交回 `Express`。以範例中的 `new List<SchoolDAO>() { new SchoolDAO() { Id = x.Id } }` 為例，會依序經過 ListInit → New → MemberInit → New → Member。`MemberInitExpressioner` 先建立物件，再逐一處理 `{ Id = x.Id }` 這類屬性指定：

```csharp csharp
private object ApplyMemberInit(MemberInitExpression memberInit, object source)
{
    object obj = MapExpression(memberInit.NewExpression, source);

    foreach (MemberBinding binding in memberInit.Bindings)
    {
        if (!(binding is MemberAssignment assignment))
            throw new NotSupportedException($"不支援的繫結類型: {binding.BindingType}");

        PropertyInfo property = (PropertyInfo)assignment.Member;
        object value = MapExpression(assignment.Expression, source);

        property.SetValue(obj, MapValue(value, property.PropertyType));
    }

    return obj;
}
```

`ListInitExpressioner` 則是集合初始化器的原理：先 new 出集合，再對每個初始化項目呼叫 `Add`（`initializer.AddMethod`）：

```csharp csharp
foreach (ElementInit initializer in listInit.Initializers)
{
    object[] args = new object[initializer.Arguments.Count];
    for (int i = 0; i < initializer.Arguments.Count; i++)
    {
        args[i] = MapExpression(initializer.Arguments[i], source);
    }

    initializer.AddMethod.Invoke(list, args);
}
```

寫完這個求值器後，我對「LINQ to SQL / EF 為什麼能把 Lambda 翻譯成 SQL」有了很具體的感覺：它們做的是同一件事，只是輸出從「值」換成了「SQL 字串」。

## 6. 泛型 + Expression：OrderBy2 快速排序

同樣的技巧也可以用在排序：`OrderBy2(x => x.Gender)` 從 Expression 取出 `PropertyInfo`，比較時用反射讀值，只要值實作 `IComparable` 就能排序。

```csharp csharp
public static PropertyInfo GetProperty<T, U>(this Expression<Func<T, U>> expression)
{
    MemberExpression member = (MemberExpression)expression.Body;
    return typeof(T).GetProperty(member.Member.Name);
}
```

```csharp csharp
private int Partition(T[] items, int left, int right)
{
    T part = items[right];
    int smallerIndex = left - 1;

    for (int i = left; i < right; i++)
    {
        if (Compare(items[i], part) > 0)
            continue;

        smallerIndex++;
        (items[i], items[smallerIndex]) = (items[smallerIndex], items[i]);
    }

    (items[smallerIndex + 1], items[right]) = (items[right], items[smallerIndex + 1]);
    return smallerIndex + 1;
}
```

`QuickSorter<T>.OrderBy<TKey>` 回傳自己，所以可以寫成 `new QuickSorter<T>().OrderBy(expression).Sort(enumerable)` 的鏈式呼叫。交換兩個元素用的是 Tuple 解構，不需要暫存變數。

# 待改進

1. **`BinaryExpressioner` 只支援 int 與 bool**：`Expressions/BinaryExpressioner.cs` 的 `ApplyBinaryExpression` 把運算元一律強轉成 `(int)` 或 `(bool)`。`object` 拆箱必須是完全相同的型別，所以 `x.Remain + 1f` 這種 `float` 運算，或 `x.Name == "abc"` 這種字串比較，都會在拆箱時丟出 `InvalidCastException`；`Subtract`、`GreaterThan` 等其他運算子則直接丟 `NotSupportedException`。修正方向：像 `UnaryExpressioner` 處理 `Increment` 那樣改用 `dynamic` 運算，或乾脆把 `ForMember` 的來源 Lambda 編譯成委派直接執行。
2. **來源沒有同名屬性時會 NullReferenceException**：`Converters/ClassConverter.cs` 對目標的每個屬性呼叫 `source.GetType().GetProperty(property.Name)`，找不到時回傳 null，下一行的 `sourceProp.GetValue(source)` 就會 NullReferenceException。也就是目標類別只要多一個來源沒有的屬性，整個轉換就失敗，連 `ForMember` 都來不及補。應該在 `sourceProp == null` 時跳過（`continue`），讓 `ForMember` 處理。
3. **`QuickSorter.Sort` 在 0 或 1 筆資料時回傳 null**：`Sorters/QuickSorter.cs` 的 `QuickSort` 在 `left >= right` 時回傳 `null`，而 `Sort` 直接回傳最外層呼叫的結果。集合只有 0 或 1 個元素時，最外層就符合這個條件，`OrderBy2` 會回傳 null 而不是陣列，呼叫端存取時就會 NullReferenceException。修正方向：`QuickSort` 改成 `void`，`Sort` 一律回傳排序後的陣列。
4. **`MemberExpressioner` 忽略了成員所屬的物件**：`Expressions/MemberExpressioner.cs` 直接 `((PropertyInfo)member.Member).GetValue(source)`，沒有先對 `member.Expression` 求值。只有 `x.Id` 這種「直接取參數屬性」的情況是對的；`x.Name.Length` 會拿 `string.Length` 去讀 `UserDTO` 物件而丟出例外，Lambda 若引用外部區域變數（編譯器產生的 closure 欄位是 `FieldInfo` 而非 `PropertyInfo`）則會在強轉時失敗。修正方向：先 `MapExpression(member.Expression, source)` 取得實際物件，再依 `PropertyInfo` / `FieldInfo` 分別讀值。

# 小結

這個專案讓我把「反射」和「Expression Tree」從聽過變成真的會用：用命名慣例加 `Activator.CreateInstance` 做分派、用遞迴處理巢狀型別、用 `Expression<Func<>>` 讀出 Lambda 的結構再自己求值。它離真正的 AutoMapper 還很遠（沒有快取、每次轉換都在做反射），但先自己造過一次輪子，回頭看套件的 API 設計就清楚多了。

---

- 系列目錄：{% post_link csharp/csharp-course-review 'C# 課程實作總整理' %}
- 上一篇：{% post_link csharp/projects/accounting-app '【C# 實作 06】AccountingApp' %}
- 下一篇：{% post_link csharp/basics/parallel-processing '【C# 實作 08】ParallelProcessing' %}
