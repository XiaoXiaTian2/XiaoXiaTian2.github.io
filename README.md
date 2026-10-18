# George 的個人部落格

使用 [Hexo](https://hexo.io/) 8 與 [Shoka](https://github.com/amehime/hexo-theme-shoka) 主題，部署到 <https://XiaoXiaTian2.github.io>。

- `dev` 分支：部落格原始碼（本 repo）
- `main` 分支：`hexo deploy` 產生的靜態網站

## 環境準備

```bash
npm install
```

## 新增文章

### 一般文章

```bash
npx hexo new "文章檔名"
```

檔案會建立在 `source/_posts/文章檔名.md`，再到 front-matter 補上 `categories`、`tags`、`cover`。

### C# 系列文

使用 `scaffolds/csharp.md` 範本，並用 `-p` 指定放在哪個子分類資料夾：

```bash
npx hexo new csharp "文章標題" -p csharp/<子分類資料夾>/<檔名>
```

範例：

```bash
npx hexo new csharp "泛型筆記" -p csharp/basics/generics
```

建立後：

1. 把 front-matter 的子分類改成與資料夾一致（見下方對照表）
2. 從固定詞彙挑選標籤（見下方標籤規範）
3. 系列文的文末加上導覽連結，並在總整理文章 `source/_posts/csharp/csharp-course-review.md` 的「系列文目錄」補上新文章：

   ```markdown
   - 系列目錄：{% post_link csharp/csharp-course-review 'C# 課程實作總整理' %}
   - 上一篇：{% post_link csharp/basics/automapper '【C# 實作 07】AutoMapper' %}
   ```

   `post_link` 的路徑是 `source/_posts/` 下的相對路徑，不含 `.md`。

### 分類規範

C# 文章使用兩層分類：`C#` > 子分類。資料夾名稱即分類網址。

| 子分類 | 資料夾 / 網址 | 內容 |
| --- | --- | --- |
| 語言基礎 | `csharp/basics/` | 泛型、委派、LINQ、反射、非同步等語言特性 |
| 資料結構 | `csharp/data-structure/` | 自製資料結構與演算法 |
| 設計模式 | `csharp/design-pattern/` | 設計模式的實作 |
| 桌面應用 | `csharp/desktop/` | WinForms、WPF |
| 專案實作 | `csharp/projects/` | 綜合型專案 |

新增子分類時，要在 `_config.yml` 的 `category_map` 加上中文名稱到英文網址的對應，否則網址會是中文。

### 標籤規範

從固定詞彙挑選，新增詞彙前先確認沒有同義的標籤：

| 層級 | 標籤 |
| --- | --- |
| 語言 / 平台 | `C#`、`.NET` |
| 知識點 | `泛型`、`委派`、`LINQ`、`反射`、`Expression Tree`、`非同步`、`資料結構`、`設計模式` |
| 技術 | `WinForms`、`WPF`、`EF Core` |
| 文章類型 | `學習筆記`、`專案實作`、`踩坑紀錄` |

含特殊符號的標籤（例如 `C#`、`.NET`）要在 `_config.yml` 的 `tag_map` 加上英文網址對應。

### 封面圖

- 文章封面：front-matter 的 `cover`，例如 `cover: assets/sharing.jpg`（圖片放在 `source/assets/`）
- 分類封面：Shoka 只有在 `source/_posts/<分類網址>/cover.jpg` 存在時，才會在首頁顯示該分類卡片，例如 `source/_posts/csharp/cover.jpg`、`source/_posts/csharp/basics/cover.jpg`

### 程式碼區塊

程式碼區塊的開頭要同時寫語言與標題，格式是 ` ```語言 標題 `：

````markdown
```csharp csharp
var total = items.Sum(x => x.Price);
```
````

- 沒有特定語言時寫 ` ```text text `
- 標題可以是任意文字，例如 ` ```csharp Program.cs `

本站由 Hexo 內建的 highlight.js 處理程式碼區塊，只有寫了標題才會產生標題列。少了標題列，Shoka 左上角的三個圓點會疊在第一行程式碼上。

## 預覽與部署

```bash
npx hexo clean       # 清除快取與 public/
npx hexo server      # 本機預覽 http://localhost:4000
npx hexo deploy -g   # 產生網站並部署到 main 分支
```
