---
layout: post
title: "PostgreSQL 資料庫遷移實戰筆記"
date: 2024-11-14
categories: [Database, DevOps]
tags: [PostgreSQL, Database Migration, pg_dump, Data Migration]
author: Allen Liao
---

## 目的

記錄 PostgreSQL 資料庫間的資料遷移流程，特別針對目標資料庫已透過 migration 建立 schema 的情境。本文詳細說明正確的遷移方法、容易遇到的問題，以及完整的操作步驟。

## 核心重點

### 1. 遷移情境

- **來源**：model DB（舊資料庫）
- **目標**：aiservice DB（新資料庫，schema 已透過 migration 建立）
- **關鍵限制**：目標 DB 的 schema 不應被覆蓋

### 2. 正確方法

使用 `pg_dump --data-only` 只遷移資料，保留目標 DB 的既有 schema 結構。

### 3. 必要的後處理

- 重置 sequence（避免 primary key 衝突）
- 驗證資料完整性
- 更新資料庫統計資訊

### 4. 權限管理

確保 table owner 與應用程式使用的 user 一致，避免 migration 執行失敗。

## 背景說明

### 遷移情境

- **來源 DB**：model
- **目標 DB**：aiservice
- **特殊狀況**：aiservice 已經透過 Go migration 完成 schema 初始化
  - xorm `Sync2()` 建立 tables
  - migration scripts 建立 indexes、constraints、triggers

### 使用者權限設定

aiservice DB 有兩個使用者帳號：

- **aiservice**：應用程式專用帳號（可能透過 trust 或 peer 認證）
- **postgres**：超級使用者（有密碼保護）

這符合最佳實踐：應用程式使用專用帳號，避免過高權限造成的安全風險。

## 方法比較

### GUI 工具 (TablePlus) vs 命令列工具 (pg_dump)

| 項目 | TablePlus Export | pg_dump --data-only |
|------|------------------|---------------------|
| CREATE TABLE | ✅ 包含（會覆蓋現有 schema） | ❌ 不含 |
| ALTER OWNER | ✅ 包含（會改成執行者） | ❌ 不含 |
| CREATE SEQUENCE | ✅ 包含 | ❌ 不含 |
| INSERT data | ✅ 包含 | ✅ 包含 |
| Triggers/Constraints | ✅ 可能包含 | ❌ 不含 |
| 適用情境 | 全新空白資料庫 | Schema 已存在的資料庫 |
| 控制程度 | 低（依賴 GUI 設定） | 高（精確控制參數） |

### TablePlus 匯出的問題

TablePlus 預設匯出包含完整的資料庫結構：

```sql
-- TablePlus Export 產生的內容
DROP TABLE IF EXISTS users;

CREATE TABLE users (
    id integer NOT NULL,
    name varchar(255),
    email varchar(255),
    created_at timestamp
);

CREATE SEQUENCE users_id_seq ...;

ALTER TABLE users OWNER TO postgres;      -- ⚠️ 覆蓋 owner
ALTER SEQUENCE users_id_seq OWNER TO postgres;  -- ⚠️ 覆蓋 owner

INSERT INTO users VALUES (...);
```

**問題點**：

1. 覆蓋目標 DB 已建立的 schema
2. Table owner 變更為執行 psql 的使用者
3. 導致應用程式 migration 執行失敗

**錯誤訊息範例**：

```
ERROR: must be owner of table users
```

原因：應用程式使用 aiservice 帳號連線，但 table owner 是 postgres，因此無法執行 ALTER TABLE 操作。

## 正確的遷移流程

### Step 1: 匯出純資料

使用 `pg_dump` 搭配 `--data-only` 參數：

```bash
pg_dump -h <model_host> -p <model_port> -U <user> \
  --data-only \
  --disable-triggers \
  -d model_db \
  -f model_data.sql
```

**參數說明**：

- `--data-only`：只匯出 INSERT 語句，不含 DDL（CREATE TABLE 等）
- `--disable-triggers`：匯入時暫時停用 trigger，避免連鎖觸發問題

**產生的 SQL 內容**：

```sql
SET session_replication_role = replica;

INSERT INTO users VALUES (1, 'Allen', 'allen@example.com', '2024-01-01');
INSERT INTO users VALUES (2, 'Bob', 'bob@example.com', '2024-01-02');

SET session_replication_role = DEFAULT;
```

### Step 2: 匯入資料

```bash
# 使用 postgres 使用者匯入（需要 superuser 權限處理 triggers）
psql -h localhost -p 54332 -U postgres -d aiservice -f model_data.sql
```

**為什麼使用 postgres 而非 aiservice？**

- `--disable-triggers` 需要 superuser 權限
- 確保匯入過程不受權限限制

### Step 3: 後處理（關鍵步驟）

#### 3.1 重置 Sequence

最重要的步驟，避免 primary key 衝突：

```sql
DO $$
DECLARE
    seq_record RECORD;
BEGIN
    FOR seq_record IN
        SELECT
            pg_get_serial_sequence(table_name::text, column_name::text) as seq_name,
            table_name,
            column_name
        FROM information_schema.columns
        WHERE table_schema = 'public'
          AND column_default LIKE 'nextval%'
    LOOP
        EXECUTE format(
            'SELECT setval(%L, (SELECT COALESCE(MAX(%I), 1) FROM %I))',
            seq_record.seq_name,
            seq_record.column_name,
            seq_record.table_name
        );
        RAISE NOTICE 'Reset sequence: %', seq_record.seq_name;
    END LOOP;
END $$;
```

**為什麼必須重置？**

若忘記重置，新增資料時會出現錯誤：

```go
user := &User{Name: "Charlie"}
err := db.Insert(user)
// ERROR: duplicate key value violates unique constraint "users_pkey"
// DETAIL: Key (id)=(1) already exists.
```

原因：sequence 值未更新，仍從初始值開始，但該 ID 已存在。

#### 3.2 驗證資料完整性

```sql
-- 檢查各 table 資料筆數
SELECT 'users' as table_name, COUNT(*) FROM users
UNION ALL
SELECT 'orders', COUNT(*) FROM orders;

-- 與來源 DB 比對，確認資料無遺漏
```

#### 3.3 更新統計資訊

```sql
-- 讓查詢優化器了解新的資料分佈
ANALYZE;
```

PostgreSQL 需要統計資訊來產生最佳查詢計畫，大量資料匯入後應更新。

## 完整操作 Checklist

### 事前準備

- [ ] 確認目標 DB schema 已透過 migration 建立
- [ ] 備份目標 DB
- [ ] 在 staging 環境測試完整流程
- [ ] 確認磁碟空間充足

### 執行流程

```bash
# 1. 匯出來源資料
pg_dump -h <model_host> -p <model_port> -U <user> \
  --data-only \
  --disable-triggers \
  -d model_db \
  -f model_data.sql

# 2. Port forward 目標 DB（如需要）
kubectl port-forward svc/postgres 54332:5432

# 3. 匯入資料
psql -h localhost -p 54332 -U postgres -d aiservice \
  -f model_data.sql

# 4. 執行後處理
psql -h localhost -p 54332 -U postgres -d aiservice << 'EOF'

-- 驗證筆數
\echo '=== 驗證資料筆數 ==='
SELECT 'users' as table_name, COUNT(*) FROM users
UNION ALL
SELECT 'orders', COUNT(*) FROM orders;

-- 重置 sequences
\echo '=== 重置 Sequences ==='
DO $$
DECLARE seq_record RECORD;
BEGIN
    FOR seq_record IN
        SELECT
            pg_get_serial_sequence(table_name::text, column_name::text) as seq_name,
            table_name, column_name
        FROM information_schema.columns
        WHERE table_schema = 'public' AND column_default LIKE 'nextval%'
    LOOP
        EXECUTE format('SELECT setval(%L, (SELECT COALESCE(MAX(%I), 1) FROM %I))',
                      seq_record.seq_name, seq_record.column_name, seq_record.table_name);
        RAISE NOTICE 'Reset: %', seq_record.seq_name;
    END LOOP;
END $$;

-- 更新統計資訊
\echo '=== 更新統計資訊 ==='
ANALYZE;

-- 驗證 owner
\echo '=== 驗證 Owner ==='
SELECT tablename, tableowner FROM pg_tables WHERE schemaname = 'public';

\echo '=== 完成 ==='
EOF
```

### 驗證測試

- [ ] Sequence 重置成功
- [ ] 資料筆數正確
- [ ] Table owner 正確（應為 aiservice）
- [ ] 測試應用程式 CRUD 功能
- [ ] Migration 能正常執行
- [ ] 檢查應用程式 logs

## 進階技巧

### 大量資料的效能優化

使用 custom format 搭配平行處理：

```bash
# 匯出（custom format）
pg_dump -h <model_host> -U <user> \
  --data-only \
  --format=custom \
  -d model_db \
  -f model_data.dump

# 匯入（4 個平行 job）
pg_restore -h localhost -p 54332 -U postgres \
  --data-only \
  --disable-triggers \
  --jobs=4 \
  -d aiservice \
  model_data.dump
```

### 關於 --column-inserts 參數

**位置式 INSERT（預設）**：

```sql
INSERT INTO users VALUES (1, 'Allen', 'allen@example.com', '2024-01-01');
```

**具名式 INSERT（加上 --column-inserts）**：

```sql
INSERT INTO users (id, name, email, created_at)
VALUES (1, 'Allen', 'allen@example.com', '2024-01-01');
```

**使用時機**：

- ✅ **建議加上**：兩邊 schema 欄位順序可能不同
- ❌ **可以不加**：確定 schema 完全一致且資料量大（具名式較慢）

本案例中，因 xorm `Sync2` 保證 schema 一致，可不加此參數，但建議先測試確認。

### 修復錯誤的 Owner

若不慎使用錯誤方法導致 owner 變更：

```sql
-- 批次修正 owner
DO $$
DECLARE r RECORD;
BEGIN
    -- 修正 tables
    FOR r IN
        SELECT tablename
        FROM pg_tables
        WHERE schemaname = 'public'
          AND tableowner = 'postgres'
    LOOP
        EXECUTE format('ALTER TABLE %I OWNER TO aiservice', r.tablename);
        RAISE NOTICE 'Changed table: %', r.tablename;
    END LOOP;

    -- 修正 sequences
    FOR r IN
        SELECT sequence_name
        FROM information_schema.sequences
        WHERE sequence_schema = 'public'
    LOOP
        EXECUTE format('ALTER SEQUENCE %I OWNER TO aiservice', r.sequence_name);
        RAISE NOTICE 'Changed sequence: %', r.sequence_name;
    END LOOP;
END $$;
```

## 常見問題

### Q1: 為什麼不能用 aiservice 使用者匯入？

`--disable-triggers` 需要 superuser 權限。若確定不需處理 triggers，可使用 aiservice，但使用 postgres 更保險。

### Q2: TablePlus 能否正確匯出純資料？

可以，但需調整設定：

1. Export 時選擇 "Data Only" 或 "INSERT statements only"
2. 取消勾選 "Include CREATE/DROP statements"
3. 匯入前檢查 SQL 內容

或手動過濾：

```bash
grep "^INSERT" exported.sql > clean_data.sql
```

### Q3: 忘記重置 sequence 如何補救？

執行重置 sequence 腳本即可，任何時候都可執行。

### Q4: Schema 不完全一致怎麼辦？

- **欄位順序不同**：加上 `--column-inserts`
- **欄位數量或型別不同**：需撰寫 migration script 轉換資料

### Q5: 能否直接在 production 執行？

強烈建議先在 staging 測試：

1. 驗證完整流程
2. 確認應用程式功能
3. 記錄執行時間
4. 準備 rollback plan
5. 選擇低流量時段執行

## 總結

### 關鍵原則

- **使用正確工具**：`pg_dump --data-only` 專門處理資料遷移
- **保護既有結構**：不覆蓋目標 DB 已建立的 schema
- **重置 sequence**：最容易忽略但最重要的步驟
- **驗證完整性**：確保資料無遺漏且權限正確
- **測試先行**：staging 環境完整驗證後再上 production

### 流程摘要

```
準備 → 確認 schema、備份、測試環境
  ↓
匯出 → pg_dump --data-only
  ↓
匯入 → psql 執行 SQL
  ↓
後處理 → 重置 sequence、驗證資料、更新統計
  ↓
驗證 → 測試應用程式功能
```

本文記錄的方法適用於目標資料庫已建立 schema 的遷移情境，透過正確的工具和完整的後處理步驟，可確保資料遷移的成功與穩定。

## 參考資料

- [PostgreSQL 官方文件 - pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)
- [PostgreSQL 官方文件 - pg_restore](https://www.postgresql.org/docs/current/app-pgrestore.html)
