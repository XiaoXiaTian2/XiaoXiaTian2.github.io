---
title: 2024六角學院後端工程師-資料庫體驗營任務二
cover: assets/sharing.jpg
date: 2024-12-01 21:10:26
categories:
  - 六角學院

tags:
  - 個人
  - 作業
  - hexschool
---

# 任務二 - 健身教練線上直播課平台

## 資料庫建立
<details>
   <summary>建立資料表</summary>

```sql sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE TABLE "USER" (
  "id" uuid PRIMARY KEY NOT NULL DEFAULT (gen_random_uuid()),
  "name" varchar(50) NOT NULL,
  "email" varchar(320) UNIQUE NOT NULL,
  "role" varchar(20) NOT NULL,
  "created_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP),
  "updated_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP)
);
CREATE TABLE "SKILL" (
  "id" uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  "name" varchar(50) UNIQUE NOT NULL,
  "created_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP)
);

INSERT INTO "SKILL" (name) VALUES ('重訓'), ('瑜伽'), ('有氧運動'), ('復健訓練');

CREATE TABLE "COACH" (
  "id" uuid PRIMARY KEY NOT NULL DEFAULT uuid_generate_v4(),
  "user_id" uuid NOT NULL REFERENCES "USER"(id),
  "experience_years" integer,
  "description" text,
  "profile_image_url" varchar(2048),
  "created_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP),
  "updated_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP),
  UNIQUE("user_id")
);

CREATE TABLE "COACH_LINK_SKILL" (
  "coach_id" uuid NOT NULL REFERENCES "COACH"(id),
  "skill_id" uuid NOT NULL REFERENCES "SKILL"(id),
  "created_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP),
  PRIMARY KEY ("coach_id", "skill_id")
);

CREATE TABLE "CREDIT_PACKAGE" (
  "id" serial PRIMARY KEY,
  "name" varchar(50) NOT NULL,
  "credit_amount" integer NOT NULL,
  "price" numeric(10,2) NOT NULL,
  "created_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP)
);

CREATE TABLE "CREDIT_PURCHASE" (
  "id" uuid PRIMARY KEY NOT NULL DEFAULT (gen_random_uuid()),
  "user_id" uuid NOT NULL REFERENCES "USER"(id),
  "credit_package_id" integer NOT NULL REFERENCES "CREDIT_PACKAGE"(id),
  "purchased_credits" integer NOT NULL,
  "price_paid" numeric(10,2) NOT NULL,
  "created_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP),
  "purchase_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP)
);

CREATE TABLE "COURSE" (
  "id" serial PRIMARY KEY,
  "user_id" uuid NOT NULL REFERENCES "USER"(id),
  "skill_id" uuid NOT NULL REFERENCES "SKILL"(id),
  "name" varchar(100) NOT NULL,
  "description" text,
  "start_at" timestamp NOT NULL,
  "end_at" timestamp NOT NULL,
  "max_participants" integer NOT NULL,
  "meeting_url" varchar(2048) NOT NULL,
  "created_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP)
);

CREATE TABLE "COURSE_BOOKING" (
  "id" uuid PRIMARY KEY NOT NULL DEFAULT (gen_random_uuid()),
  "user_id" uuid NOT NULL REFERENCES "USER"(id),
  "course_id" integer NOT NULL REFERENCES "COURSE"(id),
  "booking_at" timestamp NOT NULL,
  "status" varchar(20) NOT NULL,
  "join_at" timestamp,
  "leave_at" timestamp,
  "cancelled_at" timestamp,
  "cancellation_reason" varchar(255),
  "created_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP)
);

CREATE TABLE "BLOG_POST" (
  "id" uuid PRIMARY KEY NOT NULL DEFAULT (gen_random_uuid()),
  "user_id" uuid NOT NULL REFERENCES "USER"(id),
  "title" varchar(255) NOT NULL,
  "content" text NOT NULL,
  "featured_image_url" varchar(2048),
  "category" varchar(20) NOT NULL,
  "spend_minutes" smallint NOT NULL,
  "created_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP),
  "updated_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP)
);

CREATE TABLE "COMMENT" (
  "id" uuid PRIMARY KEY NOT NULL DEFAULT (gen_random_uuid()),
  "blog_post_id" uuid NOT NULL REFERENCES "BLOG_POST"(id),
  "user_id" uuid NOT NULL REFERENCES "USER"(id),
  "content" text NOT NULL,
  "created_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP),
  "updated_at" timestamp NOT NULL DEFAULT (CURRENT_TIMESTAMP)
);
```
</details>

## 題目及解答
### Task1 - 資料表為 USER
1-1. 新增：新增六筆用戶資料，資料如下：
- 用戶名稱為`李燕容`，email 為`lee2000@hexschooltest.io`，role為`USER`
- 用戶名稱為`王小明`，email 為`wXlTq@hexschooltest.io`，role為`USER`
- 用戶名稱為`肌肉棒子`，email 為`muscle@hexschooltest.io`，role為`USER`
- 用戶名稱為`好野人`，email 為`richman@hexschooltest.io`，role為`USER`
- 用戶名稱為`Q太郎`，email 為`starplatinum@hexschooltest.io`，role為`USER`
- 用戶名稱為 `透明人`，email 為 `opacity0@hexschooltest.io`，`role`為 `USER`
```sql sql
INSERT INTO "USER" (name, email, role)
VALUES
  ('李燕容', 'lee2000@hexschooltest.io', 'USER'),
  ('王小明', 'wXlTq@hexschooltest.io', 'USER'),
  ('肌肉棒子', 'muscle@hexschooltest.io', 'USER'),
  ('好野人', 'richman@hexschooltest.io', 'USER'),
  ('Q太郎', 'starplatinum@hexschooltest.io', 'USER'),
  ('透明人', 'opacity0@hexschooltest.io', 'USER');
```
1-2. 修改：用 email 找到 李燕容、肌肉棒子、Q太郎，如果他的 `Role` 為 `USER` 將他的 `Role` 改為 `COACH`
```sql sql
UPDATE "USER"
SET role = 'COACH'
WHERE 
  email IN ('lee2000@hexschooltest.io', 'muscle@hexschooltest.io', 'starplatinum@hexschooltest.io')
  AND 
  role = 'USER';
```
1-3. 刪除：刪除 USER 資料表中，用 email 找到透明人，並刪除該筆資料
```sql sql
DELETE FROM "USER"
WHERE email = 'opacity0@hexschooltest.io';
```
1-4. 查詢：取得 USER 資料表目前所有用戶數量（提示：使用count函式）
```sql sql
SELECT COUNT(*) AS 所有用戶數量
FROM "USER";
```
1-5. 查詢：取得 USER 資料表所有用戶資料，並列出前 3 筆（提示：使用limit語法）
```sql sql
SELECT *
FROM "USER"
LIMIT 3;
```

---
### Task2 - 資料表 組合包方案 CREDIT_PACKAGE、客戶購買課程堂數 CREDIT_PURCHASE
2-1. 新增：在`CREDIT_PACKAGE` 資料表新增三筆資料，資料需求如下：
- 名稱為 `7 堂組合包方案`，價格為`1,400` 元，堂數為`7`
- 名稱為`14 堂組合包方案`，價格為`2,520` 元，堂數為`14`
- 名稱為 `21 堂組合包方案`，價格為`4,800` 元，堂數為`21`
```sql sql
INSERT INTO "CREDIT_PACKAGE" (name, credit_amount, price)
VALUES 
  ('7 堂組合包方案', 7, 1400),
  ('14 堂組合包方案', 14, 2520),
  ('21 堂組合包方案', 21, 4800);
```
2-2. 新增：在 `CREDIT_PURCHASE` 資料表，新增三筆資料：（請使用 name 欄位做子查詢）
- `王小明` 購買 `14 堂組合包方案`
- `王小明` 購買 `21 堂組合包方案`
- `好野人` 購買 `14 堂組合包方案`

```sql sql
INSERT INTO "CREDIT_PURCHASE" (user_id, credit_package_id, purchased_credits, price_paid)
VALUES
  (
    (SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io'),
    (SELECT id FROM "CREDIT_PACKAGE" WHERE name = '14 堂組合包方案'),
    (SELECT credit_amount FROM "CREDIT_PACKAGE" WHERE name = '14 堂組合包方案'),
    (SELECT price FROM "CREDIT_PACKAGE" WHERE name = '14 堂組合包方案')
  ),
  (
    (SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io'),
    (SELECT id FROM "CREDIT_PACKAGE" WHERE name = '21 堂組合包方案'),
    (SELECT credit_amount FROM "CREDIT_PACKAGE" WHERE name = '21 堂組合包方案'),
    (SELECT price FROM "CREDIT_PACKAGE" WHERE name = '21 堂組合包方案')
  ),
  (
    (SELECT id FROM "USER" WHERE email = 'richman@hexschooltest.io'),
    (SELECT id FROM "CREDIT_PACKAGE" WHERE name = '14 堂組合包方案'),
    (SELECT credit_amount FROM "CREDIT_PACKAGE" WHERE name = '14 堂組合包方案'),
    (SELECT price FROM "CREDIT_PACKAGE" WHERE name = '14 堂組合包方案')
  );
```
- 優化版

```sql sql
WITH Users AS (
  SELECT id AS user_id, name 
  FROM "USER"
  WHERE email IN ('wXlTq@hexschooltest.io', 'richman@hexschooltest.io')
),
Package AS (
  SELECT id AS package_id, name, credit_amount, price
  FROM "CREDIT_PACKAGE"
  WHERE name IN ('14 堂組合包方案', '21 堂組合包方案')
)
INSERT INTO "CREDIT_PURCHASE" (user_id, credit_package_id, purchased_credits, price_paid)
SELECT 
  u.user_id,
  p.package_id,
  p.credit_amount,
  p.price
FROM Users u
JOIN Package p
  ON
  (u.email = 'wXlTq@hexschooltest.io' AND p.name IN ('14 堂組合包方案', '21 堂組合包方案'))
  OR
  (u.email = 'richman@hexschooltest.io' AND p.name = '14 堂組合包方案');
```

---
### Task3 - 資料表 COACH ,SKILL,COACH_LINK_SKILL
3-1. 新增：在`COACH`資料表新增三筆教練資料，資料需求如下：
- 將用戶`李燕容`新增為教練，並且年資設定為2年（提示：使用`李燕容`的email ，取得 `李燕容` 的 `id` ）
- 將用戶`肌肉棒子`新增為教練，並且年資設定為2年
- 將用戶`Q太郎`新增為教練，並且年資設定為2年
```sql sql
INSERT INTO "COACH" (user_id, experience_years)
VALUES
  (
    (SELECT id FROM "USER" WHERE email = 'lee2000@hexschooltest.io'),
    2
  ),
  (
    (SELECT id FROM "USER" WHERE email = 'muscle@hexschooltest.io'),
    2
  ),
  (
    (SELECT id FROM "USER" WHERE email = 'starplatinum@hexschooltest.io'),
    2
  );
```
- WITH優化版
```sql sql
WITH "CUSER" AS (
  SELECT id, name
  FROM "USER"
  WHERE email IN ('lee2000@hexschooltest.io', 'muscle@hexschooltest.io', 'starplatinum@hexschooltest.io')
)
INSERT INTO "COACH" (user_id, experience_years)
SELECT 
  u.id, 
  2
FROM "CUSER";
```
- 優化版
```sql sql
INSERT INTO "COACH" (user_id, experience_years)
SELECT 
  u.id,
  2
FROM "USER" u
WHERE u.email IN ('lee2000@hexschooltest.io', 'muscle@hexschooltest.io', 'starplatinum@hexschooltest.io');
```
3-2. 新增：承1，為三名教練新增專長資料至 `COACH_LINK_SKILL` ，資料需求如下：
- 所有教練都有 `重訓` 專長
- 教練`肌肉棒子` 需要有 `瑜珈` 專長
- 教練`Q太郎` 需要有 `有氧運動` 與 `復健訓練` 專長
```sql sql
INSERT INTO "COACH_LINK_SKILL" (coach_id, skill_id)
SELECT
  c.id AS coach_id,
  s.id AS skill_id
FROM "COACH" c
CROSS JOIN (SELECT id FROM "SKILL" WHERE name = '重訓') s;

INSERT INTO "COACH_LINK_SKILL" (coach_id, skill_id)
SELECT 
    c.id AS coach_id,
    s.id AS skill_id
FROM 
    "COACH" c
JOIN 
    "USER" u ON c.user_id = u.id
JOIN 
    "SKILL" s ON s.name = '瑜伽'
WHERE 
    u.email = 'muscle@hexschooltest.io';

INSERT INTO "COACH_LINK_SKILL" (coach_id, skill_id)
SELECT 
    c.id AS coach_id,
    s.id AS skill_id
FROM 
    "COACH" c
JOIN 
    "USER" u ON c.user_id = u.id
JOIN 
    "SKILL" s ON s.name IN ('有氧運動', '復健訓練')
WHERE 
    u.email = 'starplatinum@hexschooltest.io';
```
3-3. 修改：更新教練的經驗年數，資料需求如下：
- 教練`肌肉棒子` 的經驗年數為3年
- 教練`Q太郎` 的經驗年數為5年
```sql sql
SET experience_years = 
  CASE
    WHEN user_id = (SELECT id FROM "USER" WHERE email = 'muscle@hexschooltest.io') THEN 3
    WHEN user_id = (SELECT id FROM "USER" WHERE email = 'starplatinum@hexschooltest.io') THEN 5
  END;
```
- 另類寫法
```sql sql
WITH "CUser" AS (
    SELECT id AS user_id, name, email
    FROM "USER"
    WHERE email IN ('muscle@hexschooltest.io', 'starplatinum@hexschooltest.io')
)
UPDATE "COACH"
SET experience_years = 
    CASE 
        WHEN user_id = (SELECT user_id FROM "CUser" WHERE email = 'muscle@hexschooltest.io') THEN 3
        WHEN user_id = (SELECT user_id FROM "CUser" WHERE email = 'starplatinum@hexschooltest.io') THEN 5
    END
WHERE user_id IN (SELECT user_id FROM "CUser");
```
3-4. 刪除：新增一個專長 空中瑜伽 至 SKILL 資料表，之後刪除此專長。
```sql sql
INSERT INTO "SKILL" (name)
VALUES
  ('空中瑜伽');

DELETE FROM "SKILL"
WHERE name = '空中瑜伽';
```

---
### Task4 - 資料集 課程管理 COURSE 、組合包方案 CREDIT_PACKAGE
4-1. 新增：在`COURSE` 新增一門課程，資料需求如下：
- 教練設定為用戶`李燕容` 
- 在課程專長 `skill_id` 上設定為「 `重訓` 」
- 在課程名稱上，設定為「`重訓基礎課`」
- 授課開始時間`start_at`設定為 `2024-11-25 14:00:00`
- 授課結束時間`end_at`設定為 `2024-11-25 16:00:00`
- 最大授課人數`max_participants` 設定為 `10`
- 授課連結設定`meeting_url`為 `https://test-meeting.test.io`

```sql sql
INSERT INTO "COURSE" (user_id, skill_id, name, start_at, end_at, max_participants, meeting_url)
VALUES
  (
    (SELECT id FROM "USER" WHERE email = 'lee2000@hexschooltest.io'),
    (SELECT id FROM "SKILL" WHERE name = '重訓'),
    '重訓基礎課',
    '2024-11-25 14:00:00',
    '2024-11-25 16:00:00',
    10,
    'https://test-meeting.test.io'
  );
```

---
### Task5 - 資料集 客戶預約與授課 COURSE_BOOKING
5-1. 新增：請在 `COURSE_BOOKING` 新增兩筆資料：
- 1. 第一筆：`王小明` 預約 `李燕容` 的課程
  - 預約人設為`王小明`
  - 預約時間`booking_at` 設為 `2024-11-24 16:00:00`
  - 狀態`status` 設定為 `即將授課`
- 2. 新增： `好野人` 預約 `李燕容` 的課程
  - 預約人設為 `好野人`
  - 預約時間`booking_at` 設為 `2024-11-24 16:00:00`
  - 狀態`status` 設定為 `即將授課`
```sql sql
INSERT INTO "COURSE_BOOKING" (user_id, course_id, booking_at, status)
VALUES
  (
    (SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io'),
    (SELECT id FROM "COURSE" WHERE user_id = (SELECT id FROM "USER" WHERE email = 'lee2000@hexschooltest.io')),
    '2024-11-24 16:00:00',
    '即將授課'
  ),
   (
    (SELECT id FROM "USER" WHERE email = 'richman@hexschooltest.io'),
    (SELECT id FROM "COURSE" WHERE user_id = (SELECT id FROM "USER" WHERE email = 'lee2000@hexschooltest.io')),
    '2024-11-24 16:00:00',
    '即將授課'
  );
```
- 優化版
```sql sql
WITH
  user_ids AS (
    SELECT
      (SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io') AS wang_id,
      (SELECT id FROM "USER" WHERE email = 'richman@hexschooltest.io') AS hao_id,
      (SELECT id FROM "USER" WHERE email = 'lee2000@hexschooltest.io') AS coach_id
  ),
  course_id AS (
    SELECT id AS course_id
    FROM "COURSE"
    WHERE user_id = (SELECT coach_id FROM user_ids)
  )
INSERT INTO "COURSE_BOOKING" (user_id, course_id, booking_at, status)
VALUES
  ((SELECT wang_id FROM user_ids), (SELECT course_id FROM course_id), '2024-11-24 16:00:00', '即將授課'),
  ((SELECT hao_id FROM user_ids), (SELECT course_id FROM course_id), '2024-11-24 16:00:00', '即將授課');
```
5-2. 修改：`王小明` 取消預約 `李燕容` 的課程，請在`COURSE_BOOKING`更新該筆預約資料：
- 取消預約時間`cancelled_at` 設為 `2024-11-24 17:00:00`
- 狀態`status` 設定為 `課程已取消`
```sql sql
UPDATE "COURSE_BOOKING"
SET
  cancelled_at = '2024-11-24 17:00:00',
  status = '課程已取消'
WHERE
  user_id = (SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io')
  AND
  course_id = (
    SELECT id FROM "COURSE" WHERE user_id = (
      SELECT id FROM "USER" WHERE email = 'lee2000@hexschooltest.io'
    )
  );
```
5-3. 新增：`王小明` 再次預約 `李燕容` 的課程，請在`COURSE_BOOKING`新增一筆資料：
- 預約人設為`王小明`
- 預約時間`booking_at` 設為 `2024-11-24 17:10:25`
- 狀態`status` 設定為 `即將授課`
```sql sql
WITH
  user_ids AS (
    SELECT 
      (SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io') AS wang_id,
      (SELECT id FROM "USER" WHERE email = 'lee2000@hexschooltest.io') AS coach_id
  ),
  course_id AS (
    SELECT id AS course_id
    FROM "COURSE"
    WHERE user_id = (SELECT coach_id FROM user_ids)
  )
INSERT INTO "COURSE_BOOKING" (user_id, course_id, booking_at, status)
VALUES
  (
    (SELECT wang_id FROM user_ids),
    (SELECT course_id FROM course_id),
    '2024-11-24 17:10:25',
    '即將授課'
  );
```
5-4. 查詢：取得王小明所有的預約紀錄，包含取消預約的紀錄
```sql sql
SELECT *
FROM "COURSE_BOOKING"
WHERE
  user_id = (SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io');
```
5-5. 修改：`王小明` 現在已經加入直播室了，請在`COURSE_BOOKING`更新該筆預約資料（請注意，不要更新到已經取消的紀錄）：
- 請在該筆預約記錄他的加入直播室時間 `join_at` 設為 `2024-11-25 14:01:59`
- 狀態`status` 設定為 `上課中`
```sql sql
UPDATE "COURSE_BOOKING"
SET
  join_at = '2024-11-25 14:01:59',
  status = '上課中'
WHERE
  user_id = (SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io')
  AND
  course_id = (SELECT id FROM "COURSE" WHERE user_id = (SELECT id FROM "USER" WHERE email = 'lee2000@hexschooltest.io'))
  AND
  status = '即將授課';
```
5-6. 查詢：計算用戶王小明的購買堂數，顯示須包含以下欄位： user_id , total。 (需使用到 SUM 函式與 Group By)
```sql sql
SELECT user_id, SUM(purchased_credits) AS total
FROM "CREDIT_PURCHASE"
WHERE user_id = (
  SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io'
)
GROUP BY user_id;
```
5-7. 查詢：計算用戶王小明的已使用堂數，顯示須包含以下欄位： user_id , total。 (需使用到 Count 函式與 Group By)
```sql sql
SELECT
  "COURSE_BOOKING".user_id,
  COUNT(*) as total
FROM "COURSE_BOOKING" 
WHERE "COURSE_BOOKING".user_id = (SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io')
  AND status != '課程已取消'
GROUP BY "COURSE_BOOKING".user_id;
```
5-8. [挑戰題] 查詢：請在一次查詢中，計算用戶王小明的剩餘可用堂數，顯示須包含以下欄位： user_id , remaining_credit
**提示：**
```sql sql
select ("CREDIT_PURCHASE".total_credit - "COURSE_BOOKING".used_credit) as remaining_credit, ...
from ( 用戶王小明的購買堂數 ) as "CREDIT_PURCHASE"
inner join ( 用戶王小明的已使用堂數 ) as "COURSE_BOOKING"
on "COURSE_BOOKING".user_id = "CREDIT_PURCHASE".user_id;
```

```sql sql
SELECT
  "CREDIT_PURCHASE".user_id as user_id,
  ("CREDIT_PURCHASE".total_credit - "COURSE_BOOKING".used_credit) AS remaining_credit
FROM (
  SELECT 
    user_id,
    SUM(purchased_credits) AS total_credit
  FROM "CREDIT_PURCHASE"
  WHERE user_id = (
    SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io'
  )
  GROUP BY user_id
) AS "CREDIT_PURCHASE"
INNER JOIN (
  SELECT
    user_id,
    COUNT(*) as used_credit
  FROM "COURSE_BOOKING" 
  WHERE user_id = (SELECT id FROM "USER" WHERE email = 'wXlTq@hexschooltest.io')
    AND status != '課程已取消'
  GROUP BY user_id
) AS "COURSE_BOOKING"
ON "COURSE_BOOKING".user_id = "CREDIT_PURCHASE".user_id;
```

---
### Task6 - 後臺報表查詢
6-1. 查詢：查詢專長為重訓的教練，並按經驗年數排序，由資深到資淺（需使用 inner join 與 order by 語法）
- 顯示須包含以下欄位： 教練名稱 , 經驗年數, 專長名稱
```sql sql
SELECT
  u.name AS "教練名稱",
  c.experience_years AS "經驗年數",
  s.name AS "專長名稱"
FROM "COACH_LINK_SKILL" cls
JOIN "COACH" c ON cls.coach_id = c.id
JOIN "SKILL" s ON cls.skill_id = s.id
JOIN "USER" u ON c.user_id = u.id
WHERE cls.skill_id = (SELECT id FROM "SKILL" WHERE name = '重訓')
ORDER BY c.experience_years DESC;
```
6-2. 查詢：查詢每種專長的教練數量，並只列出教練數量最多的專長（需使用 group by, inner join 與 order by 與 limit 語法）
- 顯示須包含以下欄位： 專長名稱, coach_total
```sql sql
SELECT
  s.name AS "專長名稱",
  COUNT(*) AS coach_total
FROM "COACH_LINK_SKILL" cls
JOIN "SKILL" s ON cls.skill_id = s.id
GROUP BY s.name
ORDER BY coach_total DESC
LIMIT 1;
```
6-3. 查詢：計算 11 月份組合包方案的銷售數量
- 顯示須包含以下欄位： 組合包方案名稱, 銷售數量
```sql sql
SELECT
  "CREDIT_PACKAGE".name AS "組合包方案名稱",
  COUNT(*) AS "銷售數量"
FROM "CREDIT_PURCHASE"
JOIN "CREDIT_PACKAGE" ON "CREDIT_PURCHASE".credit_package_id = "CREDIT_PACKAGE".id
WHERE "CREDIT_PURCHASE".created_at BETWEEN '2024-11-01 00:00:00' AND '2024-11-30 23:59:59'
GROUP BY "CREDIT_PACKAGE".name;
```
6-4. 查詢：計算 11 月份總營收（使用 purchase_at 欄位統計）
- 顯示須包含以下欄位： 總營收
```sql sql
SELECT
  SUM(price_paid) AS "總營收"
FROM "CREDIT_PURCHASE"
WHERE "CREDIT_PURCHASE".purchase_at
BETWEEN '2024-11-01 00:00:00' AND '2024-11-30 23:59:59';
```
6-5. 查詢：計算 11 月份有預約課程的會員人數（需使用 Distinct，並用 created_at 和 status 欄位統計）
- 顯示須包含以下欄位： 預約會員人數
```sql sql
SELECT
  COUNT(DISTINCT(user_id)) AS "預約會員人數"
FROM "COURSE_BOOKING"
WHERE 
  created_at BETWEEN '2024-11-01 00:00:00' AND '2024-11-30 23:59:59'
  AND
  status != '課程已取消';
```