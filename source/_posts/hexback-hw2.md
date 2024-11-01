---
title: 2024六角學院後端工程師-資料庫體驗營小節作業二
cover: assets/sharing.jpg
date: 2024-11-01 20:50:26
categories:
  - 六角學院

tags:
  - 個人
  - 作業
  - hexschool
---

# 小節作業二 - 主鍵、外來鍵、inner join

## 1. 哪個欄位可以做為外鍵

![students](/images/hexschool/hexback-hw2/number_one.png)

### ==Ans：將班級做成外鍵==

- 建立班級 (classses) 表

```sql sql
CREATE TABLE classes (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50)
);
```

- 建立學生 (students) 表並使用外鍵參考 classes 的 id

```sql sql
CREATE TABLE students (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50),
  class_id INTEGER,
  gender VARCHAR(2),
  age INTEGER,

  FOREIGN KEY (class_id) REFERENCES classes(id)
);
```

- 新增資料

```sql sql
INSERT INTO classes (name) VALUES
  ('三年一班'),
  ('三年二班');

INSERT INTO students (name, class_id, gender, age) VALUES
  ('小明', 1, '男', 8),
  ('小華', 2, '女', 9),
  ('小美', 1, '男', 8),
  ('小強', 1, '女', 8),
  ('小智', 2, '男', 9);
```

- 透過 INNER JOIN 查詢

```sql sql
SELECT
  students.id AS 學生編號,
  students.name AS 姓名,
  classes.name AS 班級,
  students.gender AS 性別,
  students.age AS 年齡
FROM students
INNER JOIN classes ON students.class_id = classes.id;
```

---

## 2. 接續第一題，但多增加 班級老師

![students2](/images/hexschool/hexback-hw2/number_two.png)

### ==同樣拆出 **班級，班級老師** 也可同樣拆成一張表==

- 建立班級老師 (teachers) 表

```sql sql
CREATE TABLE teachers (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50)
);
```

- 建立班級 (classes) 表

```sql sql
CREATE TABLE classes (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50)
);
```

- 建立學生 (students) 表，並使用外鍵參考 classes、teachers 的 id

```sql sql
CREATE TABLE students (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50),
  class_id INTEGER,
  teacher_id INTEGER,
  gender VARCHAR(2),
  age INTEGER,

  FOREIGN KEY (class_id) REFERENCES classes(id),
  FOREIGN KEY (teacher_id) REFERENCES teachers(id)
);
```

- 建立三張表的資料

```sql sql
INSERT INTO classes (name) VALUES
  ('三年一班'),
  ('三年二班');

INSERT INTO teachers (name) VALUES
  ('廖洧杰'),
  ('卡斯伯'),
  ('查理'),
  ('麥可'),
  ('李燕蓉');

INSERT INTO students (name, class_id, teacher_id, gender, age) VALUES
  ('小明', 1, 1, '男', 8),
  ('小華', 2, 2, '女', 9),
  ('小美', 1, 3, '男', 8),
  ('小強', 1, 4, '女', 8),
  ('小智', 2, 5, '男', 9);
```

- 利用 INNER JOIN 查詢結果

```sql sql
SELECT
  students.id AS 學生編號,
  students.name AS 姓名,
  classes.name AS 班級,
  teachers.name AS 班級老師,
  students.gender AS 性別,
  students.age AS 年齡
FROM students
INNER JOIN classes ON students.class_id = classes.id
INNER JOIN teachers ON students.teacher_id = teachers.id;
```

---

## 3. 家庭資料表，減少重複

![families](/images/hexschool/hexback-hw2/number_three.png)

### ==將 **父母相關資料** 皆拆出==

- 建立父母 (parents) 表

```sql sql
CREATE TABLE parents(
  id SERIAL PRIMARY KEY,
  name VARCHAR(50),
  phone VARCHAR(10),
  gender VARCHAR(2)
);
```

- 建立孩子 (children) 表，並使用外鍵參考 parents 的 id

```sql sql
CREATE TABLE children (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50),
  parent_id INTEGER,

  FOREIGN KEY (parent_id) REFERENCES parents(id)
);
```

- 新增 parents、children 資料

```sql sql
INSERT INTO parents (name, phone, gender) VALUES
  ('王大祥', '0973254254', '男'),
  ('王曉茹', '0955717855', '女');

INSERT INTO children (name, parent_id) VALUES
  ('小明', 1),
  ('小華', 2),
  ('小美', 1),
  ('小強', 2),
  ('小智', 1);
```

- 利用 INNER JOIN 查詢結果

```sql sql
SELECT
  children.id AS 小孩編號,
  children.name AS 姓名,
  parents.name AS 父母名稱,
  parents.phone AS 父母電話,
  parents.gender AS 父母性別
FROM children
INNER JOIN parents ON children.parent_id = parents.id;
```

---

### 4. 回答前一位同學題目

:::info
**延伸作業二，三年一班老師請產假，改由原三年二班老師接，校長'廖洧杰'接三年二班老師**
:::

- 根據題意假定為三年一班麥可老師請產假，改由三年二班廖洧杰老師接任
- teachers 表

```sql sql
CREATE TABLE teachers (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50)
);
```

- students 表

```sql sql
CREATE TABLE students (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50),
  teacher_id INTEGER,
  gender VARCHAR(2),
  age INTEGER,

  FOREIGN KEY (teacher_id) REFERENCES teachers(id)
);
```

```sql sql
UPDATE
  students
SET
  teacher_id = (SELECT id FROM teachers WHERE name = '廖洧杰')
WHERE
  teacher_id = (SELECT id FROM teachers WHERE name = '麥可');
```

### 情境題

- 新的學期開始，所有同學都升了一個年級且不換班，並新增欄位「已註冊」，其值為整數且預設值為 0
