# SQL for Beginners — เริ่มจากศูนย์

> **สำหรับผู้ที่ไม่มี background ทาง tech** — เข้าใจ SQL พื้นฐานเพื่อเรียน SQL Injection

---

## Table of Contents
1. [Database คืออะไร](#database-คืออะไร--คิดแบบ-excel)
2. [ทำไมต้องมี Database](#ทำไมต้องมี-database)
3. [SQL คืออะไร](#sql-คืออะไร)
4. [SQL Commands หลัก](#sql-commands-ที่ต้องรู้-5-ตัวหลัก)
5. [WHERE Conditions](#where-conditions--เงื่อนไขต่างๆ)
6. [String และ Quote](#string-ใน-sql--ต้องใส่-quote)
7. [Comments](#comments-ใน-sql--ทำให้ส่วนที่เหลือไม่ทำงาน)
8. [UNION](#union--รวม-query-2-อัน)
9. [Login Query ทำงานยังไง](#ทดลองเข้าใจ-login-query)
10. [SQL Injection เกิดยังไง](#-sql-injection-เกิดยังไง)

---

## Database คืออะไร? — คิดแบบ Excel

**Database = Excel file ที่ฉลาดขึ้น**

```
Excel Workbook        =    Database
Excel Sheet           =    Table
Column หัวตาราง       =    Column
แถวข้อมูล             =    Row (Record)
```

**ตัวอย่าง:** ร้านค้าออนไลน์เก็บข้อมูลลูกค้า

```
┌─────────────────────────────────────────────────────┐
│  Sheet: users (ตาราง users)                         │
├──────┬───────────┬──────────────┬──────────────────┤
│  id  │  username │  password    │  email           │  ← หัว Column
├──────┼───────────┼──────────────┼──────────────────┤
│  1   │  admin    │  admin123    │  admin@shop.com  │  ← Row 1
│  2   │  carlos   │  password    │  carlos@mail.com │  ← Row 2
│  3   │  wiener   │  peter       │  wiener@mail.com │  ← Row 3
└──────┴───────────┴──────────────┴──────────────────┘
```

---

## ทำไมต้องมี Database?

| ถ้าใช้ Excel | ถ้าใช้ Database |
|-------------|----------------|
| เปิดไฟล์ทีละคน | หลายคนใช้พร้อมกันได้ |
| ค้นหาช้า | ค้นหาเร็วมาก |
| ข้อมูลหาย/พัง ง่าย | มี backup, recovery |
| ไม่เชื่อมกับเว็บได้ | เว็บดึงข้อมูลได้เลย |

**สรุป:** Website ทุกอันที่มี login, สินค้า, ข้อมูลอะไรก็ตาม → ใช้ Database

---

## SQL คืออะไร?

**SQL (Structured Query Language)** = ภาษาที่ใช้คุยกับ Database

```
คุณ: "ขอดูรายชื่อ user ทั้งหมด"
SQL: SELECT * FROM users

คุณ: "หา user ที่ชื่อ admin"
SQL: SELECT * FROM users WHERE username = 'admin'

คุณ: "เพิ่ม user ใหม่"
SQL: INSERT INTO users (username, password) VALUES ('newguy', 'pass123')
```

---

## SQL Commands ที่ต้องรู้ (5 ตัวหลัก)

### 1. SELECT — ดึงข้อมูล (ใช้บ่อยสุด!)

```sql
-- ดึงทุกอย่างจาก table users
SELECT * FROM users

-- ดึงแค่บาง column
SELECT username, email FROM users
```

**แปลเป็นไทย:**
```
SELECT    = เลือก (column อะไร)
*         = ทุก column
FROM      = จาก (table ไหน)
```

---

### 2. WHERE — กรองข้อมูล

```sql
-- หา user ที่ชื่อ admin
SELECT * FROM users WHERE username = 'admin'

-- หา user ที่ id = 1
SELECT * FROM users WHERE id = 1

-- หา user ที่ชื่อ admin และ password คือ admin123
SELECT * FROM users WHERE username = 'admin' AND password = 'admin123'
```

**แปลเป็นไทย:**
```
WHERE     = ที่/โดยที่ (เงื่อนไข)
=         = เท่ากับ
AND       = และ
OR        = หรือ
```

---

### 3. INSERT — เพิ่มข้อมูล

```sql
INSERT INTO users (username, password, email)
VALUES ('newuser', 'pass123', 'new@mail.com')
```

**แปลเป็นไทย:**
```
INSERT INTO  = ใส่เข้าไปใน (table)
VALUES       = ค่าคือ
```

---

### 4. UPDATE — แก้ไขข้อมูล

```sql
-- เปลี่ยน password ของ admin
UPDATE users SET password = 'newpass' WHERE username = 'admin'
```

**แปลเป็นไทย:**
```
UPDATE    = อัพเดท (table)
SET       = ตั้งค่า (column = ค่าใหม่)
WHERE     = เฉพาะที่ (เงื่อนไข)
```

---

### 5. DELETE — ลบข้อมูล

```sql
-- ลบ user ที่ชื่อ olduser
DELETE FROM users WHERE username = 'olduser'
```

---

## WHERE Conditions — เงื่อนไขต่างๆ

| Operator | ความหมาย | ตัวอย่าง |
|----------|---------|---------|
| `=` | เท่ากับ | `WHERE id = 1` |
| `!=` | ไม่เท่ากับ | `WHERE id != 1` |
| `>` | มากกว่า | `WHERE price > 100` |
| `<` | น้อยกว่า | `WHERE price < 50` |
| `>=` | มากกว่าหรือเท่ากับ | `WHERE age >= 18` |
| `<=` | น้อยกว่าหรือเท่ากับ | `WHERE age <= 60` |
| `LIKE` | คล้ายกับ (pattern) | `WHERE name LIKE '%admin%'` |

### AND / OR — รวมเงื่อนไข

```sql
-- AND: ทั้งสองต้อง true
SELECT * FROM users WHERE username = 'admin' AND password = 'secret'
-- แปล: หา user ที่ชื่อ admin "และ" password คือ secret

-- OR: อันไหน true ก็ได้
SELECT * FROM users WHERE username = 'admin' OR username = 'carlos'
-- แปล: หา user ที่ชื่อ admin "หรือ" carlos
```

---

## String ใน SQL — ต้องใส่ Quote

```sql
-- ถูก: String ต้องอยู่ใน quote
SELECT * FROM users WHERE username = 'admin'

-- ถูก: ใช้ double quote ก็ได้ (บาง database)
SELECT * FROM users WHERE username = "admin"

-- ผิด: ไม่มี quote
SELECT * FROM users WHERE username = admin
```

**สำคัญสำหรับ SQLi:** การเล่นกับ quote นี่แหละคือจุดเริ่มต้น!

---

## Comments ใน SQL — ทำให้ส่วนที่เหลือไม่ทำงาน

```sql
-- นี่คือ comment (ไม่ถูก execute)
SELECT * FROM users -- ส่วนนี้ไม่ทำงาน

# MySQL ใช้ # ได้
SELECT * FROM users # ส่วนนี้ไม่ทำงาน
```

**ใช้ใน SQLi:** ตัด query ส่วนที่เราไม่ต้องการทิ้ง

---

## UNION — รวม query 2 อัน

```sql
SELECT username FROM users
UNION
SELECT product_name FROM products
```

**กฎ:** จำนวน column ต้องเท่ากัน!

```sql
-- ถูก: ทั้งสอง query มี 2 columns
SELECT username, password FROM users
UNION
SELECT product_name, price FROM products

-- ผิด: column ไม่เท่ากัน
SELECT username FROM users          -- 1 column
UNION
SELECT product_name, price FROM products  -- 2 columns
```

**ใช้ใน SQLi:** แอบดึงข้อมูลจาก table อื่น

---

## ทดลองเข้าใจ: Login Query

เมื่อคุณ login เว็บไซต์:

```
Username: admin
Password: password123
```

Backend ส่ง query แบบนี้:

```sql
SELECT * FROM users WHERE username = 'admin' AND password = 'password123'
```

**ถ้าเจอ user ที่ match** → login สำเร็จ
**ถ้าไม่เจอ** → login fail

---

## SQL Injection เกิดยังไง?

### ปกติ:
```
Input: admin
Query: SELECT * FROM users WHERE username = 'admin'
```

### SQLi:
```
Input: admin'--
Query: SELECT * FROM users WHERE username = 'admin'--'
                                              ↑↑↑
                                         ส่วนนี้ถูก comment ทิ้ง!
```

### ตัวอย่าง Bypass Login:
```
Input Username: admin'--
Input Password: anything (ไม่สำคัญ)

Query ที่เกิดขึ้น:
SELECT * FROM users WHERE username = 'admin'--' AND password = 'anything'
                                             ↑
                        ตั้งแต่ -- ไปถูกตัดทิ้งหมด = ไม่ต้องรู้ password!
```

---

## สรุป SQL ที่ต้องรู้สำหรับ SQLi

| Command | ทำอะไร | ใช้ใน SQLi ยังไง |
|---------|-------|-----------------|
| `SELECT` | ดึงข้อมูล | อ่าน credentials |
| `WHERE` | กรอง | Bypass conditions |
| `AND/OR` | รวมเงื่อนไข | `OR 1=1` bypass |
| `' (quote)` | ปิด string | Escape จาก string |
| `-- (comment)` | ตัด query | ตัดส่วนที่ไม่ต้องการ |
| `UNION` | รวม query | ดึงข้อมูลจาก table อื่น |

---

## Next Steps

- [01-SQL-Fundamentals.md](./01-SQL-Fundamentals.md) — SQL ละเอียดขึ้น (สำหรับคนมี background)
- [02-SQL-Injection-Detection.md](./02-SQL-Injection-Detection.md) — Detection & Basic Exploitation
