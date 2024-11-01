---
title: 2024六角學院後端工程師-資料庫體驗營小節作業一
cover: assets/sharing.jpg
date: 2024-10-26 15:50:26
categories:
  - 六角學院

tags:
  - 個人
  - 作業
  - hexschool
---

# 小節作業一 - 家具店的菜雞銷售奇遇記

## 資料集

```sql sql
CREATE TABLE products (
   name VARCHAR(100),         -- 商品名稱
   price INTEGER,            -- 原價
   discount_price INTEGER,    -- 優惠價
   stock INTEGER,            -- 庫存數量
   category VARCHAR(50),      -- 商品分類
   status VARCHAR(20)         -- 商品狀態
);

INSERT INTO products (name, price, discount_price, stock, category, status) VALUES
   ('北歐風雙人沙發', 39900, 35900, 3, '沙發', 'active'),
   ('貓抓皮L型沙發', 58900, 52900, 1, '沙發', 'active'),
   ('典雅三人座沙發', 42800, 42800, 5, '沙發', 'active'),
   ('工業風電視櫃', 5900, 4900, 12, '櫃子', 'active'),
   ('簡約書櫃', 3500, 3500, 8, '櫃子', 'active'),
   ('玄關鞋櫃', 2900, 2900, 15, '櫃子', 'active'),
   ('日式雙人床架', 12000, 11200, 6, '床架', 'active'),
   ('掀床五尺雙人床', 19900, 18900, 2, '床架', 'active'),
   ('兒童床架', 8900, 8900, 0, '床架', 'inactive'),
   ('電腦辦公椅', 4500, 3900, 20, '椅子', 'active'),
   ('餐椅四入組', 6000, 5200, 8, '椅子', 'active'),
   ('北歐風餐桌', 15800, 14800, 4, '桌子', 'active'),
   ('實木咖啡桌', 3200, 2900, 10, '桌子', 'active'),
   ('電競書桌', 8900, 8900, 0, '桌子', 'inactive');
```

## 作業解答

### 基礎練習

1. 找到北歐風雙人沙發的價格和庫存

```sql sql
SELECT name, price, discount_price, stock
FROM products
WHERE name = '北歐風雙人沙發';
```

2. 找出櫃子類且價格在 5000 以下的商品

```sql sql
SELECT *
FROM products
WHERE category = '櫃子'
   AND price < 5000
   AND status = 'active';
```

3. 確認日式雙人床架的庫存狀況

```sql sql
SELECT name, stock
FROM products
WHERE name = '日式雙人床架';
```

### AND 語法

4. 想找 4 萬以下，而且有現貨的沙發

```sql sql
SELECT name, price, discount_price, stock
FROM products
WHERE category = '沙發'
  AND price < 40000
  AND stock > 0
  AND status = 'active';
```

5. 找到沙發類且有特價（原價大於優惠價）且還有庫存的商品

```sql sql
SELECT *
FROM products
WHERE category = '沙發'
  AND price > discount_price
  AND stock > 0
  AND status = 'active';
```

### OR 語法

6. 找出櫃子類或桌子類的商品

```sql sql
SELECT *
FROM products
WHERE category = '櫃子'
  OR category = '桌子';
```

7. 北歐風雙人沙發和貓抓皮 L 型沙發的庫存狀況

```sql sql
SELECT name, stock
FROM products
WHERE name = '北歐風雙人沙發'
  OR name = '貓抓皮L型沙發';
```

### IN 語法

8. 沙發、櫃子和桌子這三種分類的商品

```sql sql
SELECT *
FROM products
WHERE category IN ('沙發', '櫃子', '桌子');
```

9. 電腦辦公椅和餐椅四入組的價格

```sql sql
SELECT name, price, discount_price
FROM products
WHERE name IN ('電腦辦公椅', '餐椅四入組');
```

### BETWEEN 語法

10. 價格 10000 到 20000 的所有商品

```sql sql
SELECT *
FROM products
WHERE price BETWEEN 10000 AND 20000;
```

11. 庫存在 5 到 15 之間的商品

```sql sql
SELECT *
FROM products
WHERE stock BETWEEN 5 AND 15;
```

### NOT IN 語法

12. 列出除了沙發和床架以外的商品

```sql sql
SELECT *
FROM products
WHERE category NOT IN ('沙發', '床架');
```

### UPDATE 語法

13. 調降北歐風雙人沙發 2000 元

```sql sql
UPDATE products
SET price = price - 2000
WHERE name = '北歐風雙人沙發';
```

14. 增加電腦辦公椅 5 庫存數量

```sql sql
UPDATE products
SET stock = stock + 5
WHERE name = '電腦辦公椅';
```

15. 清掉兒童床架和電競書桌的資料

```sql sql
DELETE FROM products
WHERE name IN ('兒童床架', '電競書桌');
```

## 回覆 編號 12 問題

- **兒童床架、電競書桌賣完了，將庫存補充至 20 件，並將狀態調整為 active**
- 解答：

```sql sql
UPDATE products
SET
  stock = 20,
  status = 'active'
WHERE name IN ('兒童床架', '電競書桌');
```

## 新情境

- **將所有未上架的商品補至 3 件，且更改為上架狀態**
- 解答：
<details>
   <summary>查看解答</summary>
   <p>
   ```sql sql
   UPDATE products
   SET
     stock = 3,
     status = 'active'
   WHERE status = 'inactive';
   ```
   </p>
</details>
