---
title: 【C# 實作 02】LinkedList：自製串列、二元搜尋樹、Dictionary 與迭代器
date: 2026-10-18 20:20:00
updated: 2026-10-18 20:20:00
categories:
  - C#
  - 資料結構
tags:
  - C#
  - 資料結構
  - 泛型
  - 學習筆記
---

這篇回顧課程中「自己造輪子」的資料結構練習：不用 `List<T>`、`Dictionary<TKey, TValue>`，從節點開始自己寫鏈結串列、二元搜尋樹和雜湊表，再替它們實作迭代器，讓 `foreach` 可以直接走訪。

<!-- more -->

# 專案簡介

LinkedList 是一個包含 4 個 Console 專案的方案，每個專案對應一次作業：

- 技術：.NET Framework 4.8、Console 應用程式
- 主題：泛型鏈結串列、二元搜尋樹（DFS / BFS）、雜湊表、`IEnumerable` / `IEnumerator`、集合初始化器
- Repo：[https://github.com/XiaoXiaTian2/LinkedList](https://github.com/XiaoXiaTian2/LinkedList)

這個 repo 沒有單元測試，`Program.cs` 裡留著很多註解掉的手動測試程式，也記錄了老師出的作業題目（例如「完成 BFS 的迭代模式（不可以汙染你的 Tree）」）。

# 架構與功能

| 專案 | 主要檔案 | 內容 |
| --- | --- | --- |
| LinkedList | `Node.cs`、`JackList.cs`、`ForwardIterator.cs` | 泛型單向鏈結串列 `JackList<T>`：Add、Remove、Insert、AddRange、RemoveRange、索引子，以及順向迭代器 |
| BinaryTree | `Tree.cs`、`Node.cs`、`DFSIterator.cs`、`BFSIterator.cs`、`Enums/TraversalMode.cs` | 整數二元搜尋樹：遞迴插入/刪除，依 `Mode` 切換兩種迭代器 |
| Dict | `JackDictionary.cs`、`Entry.cs` | 泛型雜湊表：bucket 陣列 + 鏈結法，負載因子超過 0.72 時擴容 |
| 測試專案 | `Students.cs` | 同時實作 `IEnumerable` 與 `IEnumerator`，練習集合初始化器 |

# 核心知識點

## 泛型節點與鏈結串列

節點只有資料和指向下一個節點的參考，`JackList<T>` 只記住 `Head`，其餘節點都靠 `Next` 串起來：

```csharp
internal class Node<T>
{
    public T Data;
    public Node<T> Next;
    public Node(T data) { Data = data; Next = null; }
}
```

`Add` 從 Head 一路走到尾端再接上新節點；`Remove` 則要同時記住「上一個節點」，找到目標後讓上一個節點跳過它：

```csharp
Node<T> node = Head;
Node<T> lastNode = null;
while (node != null)
{
    if (node.Data.Equals(item))
    {
        lastNode.Next = node?.Next;
        Count--;
        break;
    }
    lastNode = node;
    node = node.Next;
}
```

Head 本身的刪除在迴圈之前就先處理掉了，所以進迴圈後 `lastNode` 在命中時不會是 null。這也是最早版本（非泛型、用 `==` 比較字串）留在註解裡的寫法改良後的結果：改成泛型後不能用 `==`，必須用 `Equals`。

索引子 `this[int i]` 讓串列可以像陣列一樣用 `list[0]` 取值，底層是從 Head 數過去的 `GetNodeByIndex`，所以每次存取是 O(n)。

## 迭代器協定：讓自製集合支援 foreach

`foreach` 只要求型別有 `GetEnumerator()`，回傳的物件要有 `MoveNext()` 和 `Current`。`JackList<T>` 實作 `IEnumerable`，把走訪的狀態交給另一個物件 `ForwardIterator<T>`：

```csharp
internal class ForwardIterator<T> : IEnumerator
{
    private readonly JackList<T> JackList;

    public ForwardIterator(JackList<T> list) { JackList = list; }

    public int index = -1;

    public object Current => JackList[index];

    public bool MoveNext()
    {
        //index++;
        return ++index < JackList.Count; // 先更新index再比較長度
    }

    public void Reset()
    {
        index = -1;
    }
}
```

`index` 從 -1 開始，是因為 `foreach` 會先呼叫一次 `MoveNext()` 才讀 `Current`。把狀態放在獨立的迭代器裡，同一個串列就可以同時被多個 `foreach` 走訪而不互相干擾。

對照組是「測試專案」的 `Students`，它把 `IEnumerable` 和 `IEnumerator` 都實作在自己身上，`GetEnumerator()` 先 `Reset()` 再回傳 `this`。這個專案主要是用來理解**集合初始化器**：只要型別實作 `IEnumerable` 又有 `Add` 方法，就能這樣寫：

```csharp
Students students = new Students() { "Leo", "Allen", "Alice", "Chuck", "Jerry" };
```

編譯器會把大括號裡的每個元素轉成一次 `students.Add(...)` 呼叫，`Program.cs` 裡註解掉的 5 行 `students.Add` 就是它展開後的樣子。

## 二元搜尋樹：遞迴插入與刪除

插入時比當前節點大就往右、小就往左，遞迴到空位時建立新節點，再一路把子樹接回去：

```csharp
private Node InsertNode(Node initNode, int data)
{
    if (initNode == null)
    {
        return new Node(data);
    }

    if (data > initNode.Data)
    {
        initNode.Right = this.InsertNode(initNode.Right, data);
    }
    else if (data < initNode.Data)
    {
        initNode.Left = this.InsertNode(initNode.Left, data);
    }

    return initNode;
}
```

刪除比較麻煩，要分三種情況：沒有右子樹就回傳左子樹、沒有左子樹就回傳右子樹；兩邊都有時，用**右子樹的最小值**取代自己，再到右子樹把那個最小值刪掉：

```csharp
else
{
    if (initNode.Right == null)
    {
        return initNode.Left;
    }
    if (initNode.Left == null)
    {
        return initNode.Right;
    }

    Node target = FindMinNode(initNode.Right);
    initNode.Data = target.Data;
    initNode.Right = RemoveNode(initNode.Right, target.Data);
}
return initNode;
```

右子樹最小值一定比左子樹所有值大、又比右子樹其他值小，換上來之後整棵樹仍然符合 BST 的性質。

## 兩種走訪模式：用 Queue 實作 BFS

`Tree` 依 `Mode` 屬性決定 `GetEnumerator()` 回傳哪一種迭代器，`foreach` 那端完全不用改：

```csharp
public IEnumerator GetEnumerator()
{
    switch (Mode)
    {
        case TraversalMode.DFS:
            return new DFSIterator(this);
        case TraversalMode.BFS:
            return new BFSIterator(this);
        default:
            throw new NotImplementedException();
    }
}
```

`DFSIterator` 和串列的迭代器一樣是「索引 + 索引子」；索引子底下的 `GetNodeByIndex` 目前呼叫的是 `GetNodeByBFS`（`GetNodeByDFS` 那行被註解掉了）。BFS 的核心是佇列：取出一個節點，再把它的左右子節點排進隊伍，就會一層一層往下走：

```csharp
Queue<Node> queue = new Queue<Node>();
queue.Enqueue(Root);
int counter = 0;

while (queue.Count > 0)
{
    Node node = queue.Dequeue();
    if (counter == index)
        return node;

    counter++;
    if (node.Left != null)
        queue.Enqueue(node.Left);

    if (node.Right != null)
        queue.Enqueue(node.Right);
}
```

## 雜湊表：bucket、鏈結法與擴容

`JackDictionary<TKey, TValue>` 用 `Entry` 陣列當 bucket，每個 `Entry` 有 `next`，同一個 bucket 的資料串成鏈結串列（鏈結法處理碰撞）。決定 bucket 的方式是把 hashCode 取正數後對陣列長度取餘數：

```csharp
int num1 = key.GetHashCode() & 0x7FFFFFFF;
int index1 = num1 % buckets.Length;

var current1 = buckets[index1];
while (current1 != null)
{
    if (current1.hashCode == num1 && current1.key.Equals(key))
    {
        current1.value = val;
        return;
    }
    current1 = current1.next;
}
```

`& 0x7FFFFFFF` 是把最高位元（符號位）清掉，因為 `GetHashCode()` 可能是負數，負數取餘數會得到負的索引。比對時先比 hashCode（便宜）再比 `Equals`（確認真的是同一個 key）。

當 `(_count + 1f) / buckets.Length` 超過 0.72 時，陣列擴大成兩倍，並把舊資料依新的長度重新分配。每個 `Entry` 都存了自己的 `hashCode`，所以搬家時不用再呼叫一次 `GetHashCode()`：

```csharp
foreach (var oldEntry in buckets)
{
    var current = oldEntry;
    // 處理entry內每個next
    while (current != null)
    {
        int newIndex = current.hashCode % newSize;

        var nextEntry = current.next;

        current.next = newBuckets[newIndex];
        newBuckets[newIndex] = current;

        current = nextEntry;
    }
}
```

# 待改進

1. **`JackList.DetectCycle` 一開始就判定有環**：`slow` 和 `fast` 都從 `Head` 出發，而迴圈第一件事就是檢查 `slow == fast`，所以只要串列不是空的，第一圈就回傳 `true`。`AddRange` 在非空串列上接上新串列後一定會走到 `lastNode.Next = null` 把它還原，等於 AddRange 永遠沒效果。修正方向：先移動 `slow`、`fast` 再比較。另外 `AddRange` 在 `Head == null` 的分支只設定了 `Head`，沒有加上 `list.Count`，接完後 `Count` 仍是 0，迭代器什麼都走不到。
2. **`JackDictionary.GetValue` 只比對 hashCode**：`if (entry.hashCode == num)` 沒有像 `Add`、`SetValue` 一樣再比對 `key.Equals(key)`。兩個不同的 key 雜湊值相同（碰撞）時，會回傳先找到的那筆，也就是錯誤的值。補上 `&& entry.key.Equals(key)` 即可。
3. **`BFSIterator.Current` 改寫了 `Tree.Root`**：`Current` 的 getter 直接執行 `Tree.Root = _queue.Dequeue();`，走訪一次之後 `Root` 會停在最後一個被取出的節點，整棵樹就「不見了」，正好違反作業要求的「不可以汙染你的 Tree」。而且讀一次 `Current` 就出列一次，連續讀兩次會跳過節點。修正方向：用迭代器自己的欄位存目前節點，並把出列的動作移到 `MoveNext()`。
4. **`JackList.Insert` 的 index 超過長度時拋出 NullReferenceException**：方法上方的註解寫著「超過索引 => throw IndexoutOfRange」，但目前只檢查了 `index < 0`。當 `index > Count`（包含對空串列插入非 0 的位置）時，迴圈會走到 `node` 為 null 後再執行 `node.Next`，拋出的是 NullReferenceException。應在開頭加上 `index > Count` 的檢查。

# 小結

自己寫一次才真正理解：`foreach` 只是 `GetEnumerator` / `MoveNext` / `Current` 的語法糖、集合初始化器只是一連串 `Add`、`Dictionary` 的效率來自 bucket 與負載因子。也因為是自己寫的，重讀時發現的邊界錯誤特別多，這正好說明這類資料結構最需要單元測試來保護。

---

- 系列目錄：{% post_link csharp/csharp-course-review 'C# 課程實作總整理' %}
- 上一篇：{% post_link csharp/projects/order-system '【C# 實作 01】OrderSystem' %}
- 下一篇：{% post_link csharp/design-pattern/rx '【C# 實作 03】Rx' %}
