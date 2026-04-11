# SQL Fundamentals — พื้นฐานก่อนเข้า SQL Injection

> **เป้าหมาย:** เข้าใจ SQL และ Database เพื่อเข้าใจว่า SQL Injection ทำงานยังไง

---

## Table of Contents
1. [Database คืออะไร](#database-คืออะไร)
2. [Database Types](#database-types)
3. [Relational Database Concepts](#relational-database-concepts)
4. [SQL Language Categories](#sql-language-categories)
5. [DDL - Data Definition Language](#ddl--สร้าง-structure)
6. [DML - SELECT Deep Dive](#dml--select-ลึกขึ้น)
7. [JOINs](#joins--รวมข้อมูลจากหลาย-tables)
8. [INSERT, UPDATE, DELETE](#insert-update-delete)
9. [Subqueries](#subqueries--query-ซ้อน-query)
10. [SQL Functions](#sql-functions)
11. [information_schema](#information_schema--database-metadata)
12. [Database-Specific Differences](#database-specific-differences)
13. [Database to Web Architecture](#database-to-web--เบื้องหลังการทำงาน)

---

## Database คืออะไร?

**Database** = ที่เก็บข้อมูลแบบมีโครงสร้าง สามารถ query, update, manage ได้

**DBMS (Database Management System)** = software ที่จัดการ database

```
┌─────────────────────────────────────────────────┐
│                    DBMS                         │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐     │
│  │ Database 1│ │ Database 2│ │ Database 3│     │
│  │  - users  │ │  - orders │ │  - logs   │     │
│  │  - posts  │ │  - items  │ │  - events │     │
│  └───────────┘ └───────────┘ └───────────┘     │
│                                                 │
│  Query Engine │ Storage │ Security │ Backup    │
└─────────────────────────────────────────────────┘
```

---

## Database Types

### Relational Database (SQL)
ข้อมูลเก็บใน **tables** ที่มีความสัมพันธ์กัน

| DBMS | ใช้ที่ไหน | Port |
|------|----------|------|
| **MySQL** | WordPress, PHP apps | 3306 |
| **PostgreSQL** | Modern web apps | 5432 |
| **MSSQL** | Enterprise, .NET | 1433 |
| **SQLite** | Mobile apps, embedded | File-based |
| **Oracle** | Enterprise, Banking | 1521 |

### NoSQL Database
ไม่ใช้ table structure แบบ relational

| Type | ตัวอย่าง | เก็บแบบ |
|------|---------|--------|
| Document | MongoDB | JSON documents |
| Key-Value | Redis | key → value |
| Graph | Neo4j | Nodes & edges |

**ใน BSCP:** เน้น Relational (SQL) เป็นหลัก

---

## Relational Database Concepts

### Tables, Rows, Columns

```
Table: users
┌─────┬──────────┬──────────────┬─────────────────┐
│ id  │ username │ password     │ email           │  ← Columns (fields)
├─────┼──────────┼──────────────┼─────────────────┤
│ 1   │ admin    │ hashed_pass1 │ admin@site.com  │  ← Row (record)
│ 2   │ carlos   │ hashed_pass2 │ carlos@site.com │  ← Row (record)
│ 3   │ wiener   │ hashed_pass3 │ wiener@site.com │  ← Row (record)
└─────┴──────────┴──────────────┴─────────────────┘
        ↑
     Column
```

### Primary Key (PK)
**Unique identifier** สำหรับแต่ละ row

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,        -- ← PK: ห้ามซ้ำ, ห้าม NULL
    username VARCHAR(50),
    password VARCHAR(255)
);
```

### Foreign Key (FK)
**เชื่อมความสัมพันธ์** ระหว่าง tables

```
Table: users                    Table: orders
┌─────┬──────────┐              ┌──────────┬─────────┬────────┐
│ id  │ username │              │ order_id │ user_id │ total  │
├─────┼──────────┤              ├──────────┼─────────┼────────┤
│ 1   │ admin    │◄─────────────│ 101      │ 1       │ 500.00 │
│ 2   │ carlos   │◄─────────────│ 102      │ 2       │ 150.00 │
│ 3   │ wiener   │◄─────────────│ 103      │ 2       │ 200.00 │
└─────┴──────────┘     FK       └──────────┴─────────┴────────┘
   PK                              orders.user_id → users.id
```

---

## SQL Language Categories

```
SQL
├── DDL (Data Definition Language)    → สร้าง/แก้ structure
│   ├── CREATE
│   ├── ALTER
│   └── DROP
│
├── DML (Data Manipulation Language)  → จัดการข้อมูล
│   ├── SELECT
│   ├── INSERT
│   ├── UPDATE
│   └── DELETE
│
├── DCL (Data Control Language)       → permissions
│   ├── GRANT
│   └── REVOKE
│
└── TCL (Transaction Control)         → transactions
    ├── COMMIT
    ├── ROLLBACK
    └── SAVEPOINT
```

---

## DDL — สร้าง Structure

### CREATE TABLE
```sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2),
    category VARCHAR(50),
    stock INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Data Types

| Type | ใช้กับ | ตัวอย่าง |
|------|-------|---------|
| `INT` | ตัวเลขจำนวนเต็ม | id, stock, age |
| `VARCHAR(n)` | string ความยาวไม่เกิน n | username, email |
| `TEXT` | string ยาวมาก | description, content |
| `DECIMAL(p,s)` | ตัวเลขทศนิยม | price, salary |
| `BOOLEAN` | true/false | is_admin, is_active |
| `DATE` | วันที่ | birth_date |
| `TIMESTAMP` | วันที่+เวลา | created_at |
| `BLOB` | binary data | images, files |

### ALTER TABLE
```sql
-- เพิ่ม column
ALTER TABLE users ADD COLUMN role VARCHAR(20);

-- แก้ไข column
ALTER TABLE users MODIFY COLUMN username VARCHAR(100);

-- ลบ column
ALTER TABLE users DROP COLUMN role;
```

### DROP
```sql
DROP TABLE users;           -- ลบ table
DROP DATABASE shop_db;      -- ลบ database ทั้งก้อน
```

**🔴 อันตราย:** ถ้า SQLi ได้ และ DB user มีสิทธิ์ DROP = ลบข้อมูลทั้งหมดได้

---

## DML — SELECT ลึกขึ้น

### Basic SELECT
```sql
SELECT * FROM users;                          -- ทุก column
SELECT username, email FROM users;            -- บาง column
SELECT DISTINCT category FROM products;       -- ไม่ซ้ำ
```

### WHERE Conditions
```sql
-- Comparison
SELECT * FROM products WHERE price > 100;
SELECT * FROM products WHERE price BETWEEN 50 AND 100;
SELECT * FROM products WHERE category IN ('Electronics', 'Books');

-- Pattern Matching
SELECT * FROM users WHERE email LIKE '%@gmail.com';
SELECT * FROM products WHERE name LIKE 'iPhone%';      -- เริ่มด้วย iPhone
SELECT * FROM products WHERE name LIKE '%Pro%';        -- มี Pro อยู่ตรงไหนก็ได้

-- NULL check
SELECT * FROM users WHERE email IS NULL;
SELECT * FROM users WHERE email IS NOT NULL;
```

### LIKE Wildcards

| Wildcard | ความหมาย | ตัวอย่าง |
|----------|---------|---------|
| `%` | ตัวอักษรกี่ตัวก็ได้ (0+) | `'%admin%'` |
| `_` | ตัวอักษร 1 ตัว | `'admin_'` → admin1, adminA |

### Logical Operators
```sql
-- AND: ทั้งสองต้อง true
SELECT * FROM users WHERE username = 'admin' AND password = 'test'

-- OR: อันใดอันหนึ่ง true ก็พอ
SELECT * FROM users WHERE username = 'admin' OR username = 'carlos'

-- NOT: กลับค่า
SELECT * FROM users WHERE NOT username = 'admin'
```

### ORDER BY
```sql
SELECT * FROM products ORDER BY price ASC;           -- น้อย → มาก
SELECT * FROM products ORDER BY price DESC;          -- มาก → น้อย
SELECT * FROM products ORDER BY category, price;     -- หลาย columns
SELECT * FROM products ORDER BY 1;                   -- column แรก (ใช้ใน SQLi)
```

### LIMIT / OFFSET
```sql
SELECT * FROM products LIMIT 10;                -- แค่ 10 rows แรก
SELECT * FROM products LIMIT 10 OFFSET 20;      -- ข้าม 20, เอา 10 ถัดไป

-- MySQL syntax
SELECT * FROM products LIMIT 20, 10;            -- LIMIT offset, count
```

### GROUP BY & Aggregate Functions
```sql
-- นับจำนวน
SELECT category, COUNT(*) as total
FROM products
GROUP BY category;

-- ผลรวม, ค่าเฉลี่ย
SELECT category, SUM(price), AVG(price)
FROM products
GROUP BY category;

-- HAVING (filter หลัง GROUP BY)
SELECT category, COUNT(*) as total
FROM products
GROUP BY category
HAVING COUNT(*) > 5;
```

| Function | ทำอะไร |
|----------|-------|
| `COUNT()` | นับจำนวน |
| `SUM()` | รวมค่า |
| `AVG()` | ค่าเฉลี่ย |
| `MAX()` | ค่ามากสุด |
| `MIN()` | ค่าน้อยสุด |

---

## JOINs — รวมข้อมูลจากหลาย Tables

```
users                           orders
┌────┬─────────┐               ┌──────┬─────────┬───────┐
│ id │ username│               │ o_id │ user_id │ total │
├────┼─────────┤               ├──────┼─────────┼───────┤
│ 1  │ admin   │               │ 101  │ 1       │ 500   │
│ 2  │ carlos  │               │ 102  │ 2       │ 150   │
│ 3  │ wiener  │               │ 103  │ 2       │ 200   │
└────┴─────────┘               │ 104  │ 9       │ 300   │ ← user_id 9 ไม่มี
                               └──────┴─────────┴───────┘
```

### INNER JOIN
เอาเฉพาะ rows ที่ **match ทั้งสอง** tables

```sql
SELECT users.username, orders.total
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```
```
Result:
┌──────────┬───────┐
│ username │ total │
├──────────┼───────┤
│ admin    │ 500   │
│ carlos   │ 150   │
│ carlos   │ 200   │
└──────────┴───────┘
```

### LEFT JOIN
เอา **ทุก rows จาก table ซ้าย** + match จากขวา (ถ้าไม่มี = NULL)

```sql
SELECT users.username, orders.total
FROM users
LEFT JOIN orders ON users.id = orders.user_id;
```
```
Result:
┌──────────┬───────┐
│ username │ total │
├──────────┼───────┤
│ admin    │ 500   │
│ carlos   │ 150   │
│ carlos   │ 200   │
│ wiener   │ NULL  │  ← wiener ไม่มี orders
└──────────┴───────┘
```

### JOIN Diagram

```
        INNER JOIN              LEFT JOIN               RIGHT JOIN
     ┌─────┬─────┐           ┌─────┬─────┐           ┌─────┬─────┐
     │     │█████│           │█████│█████│           │█████│█████│
     │  A  │█████│  B        │█████│█████│  B        │  A  │█████│
     │     │█████│           │█████│█████│           │     │█████│
     └─────┴─────┘           └─────┴─────┘           └─────┴─────┘
       เฉพาะ match            A ทั้งหมด + match        match + B ทั้งหมด
```

---

## INSERT, UPDATE, DELETE

### INSERT
```sql
-- Insert single row
INSERT INTO users (username, password, email)
VALUES ('newuser', 'pass123', 'new@site.com');

-- Insert multiple rows
INSERT INTO users (username, password, email) VALUES
    ('user1', 'pass1', 'user1@site.com'),
    ('user2', 'pass2', 'user2@site.com'),
    ('user3', 'pass3', 'user3@site.com');
```

### UPDATE
```sql
-- Update specific rows
UPDATE users
SET password = 'newpassword'
WHERE username = 'admin';

-- Update multiple columns
UPDATE products
SET price = price * 0.9, stock = stock + 100
WHERE category = 'Electronics';

-- ⚠️ ไม่มี WHERE = update ทุก rows!
UPDATE users SET is_admin = true;    -- ทุกคนเป็น admin!
```

### DELETE
```sql
-- Delete specific rows
DELETE FROM users WHERE username = 'olduser';

-- Delete with condition
DELETE FROM orders WHERE created_at < '2024-01-01';

-- ⚠️ ไม่มี WHERE = ลบทุก rows!
DELETE FROM users;    -- ลบ users ทั้งหมด!
```

---

## Subqueries — Query ซ้อน Query

```sql
-- หา users ที่มี orders
SELECT * FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders);

-- หาสินค้าที่ราคาสูงกว่าค่าเฉลี่ย
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- Subquery ใน FROM (derived table)
SELECT category, avg_price
FROM (
    SELECT category, AVG(price) as avg_price
    FROM products
    GROUP BY category
) AS category_stats
WHERE avg_price > 100;
```

---

## SQL Functions

### String Functions

| Function | ทำอะไร | ตัวอย่าง |
|----------|-------|---------|
| `CONCAT()` | ต่อ string | `CONCAT('Hello', ' ', 'World')` |
| `SUBSTRING()` | ตัด string | `SUBSTRING('Hello', 1, 3)` → 'Hel' |
| `LENGTH()` | ความยาว | `LENGTH('Hello')` → 5 |
| `UPPER()` | ตัวใหญ่ | `UPPER('hello')` → 'HELLO' |
| `LOWER()` | ตัวเล็ก | `LOWER('HELLO')` → 'hello' |
| `TRIM()` | ตัด whitespace | `TRIM('  hi  ')` → 'hi' |
| `REPLACE()` | แทนที่ | `REPLACE('hello', 'l', 'x')` → 'hexxo' |

### ใช้ใน SQLi:
```sql
-- Extract ทีละตัว (Blind SQLi)
SELECT SUBSTRING(password, 1, 1) FROM users WHERE username = 'admin';
-- → 's' (ตัวแรกของ password)

SELECT SUBSTRING(password, 2, 1) FROM users WHERE username = 'admin';
-- → 'e' (ตัวที่สอง)
```

---

## information_schema — Database Metadata

ทุก database มี **information_schema** เก็บข้อมูลเกี่ยวกับตัวเอง

### ดู Databases ทั้งหมด
```sql
SELECT schema_name FROM information_schema.schemata;
```

### ดู Tables ทั้งหมด
```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'shop_db';
```

### ดู Columns ของ Table
```sql
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'users';
```

**🔴 ใน SQLi:** นี่คือวิธีที่ attacker enumerate database structure

---

## Database-Specific Differences

### เอา Version

| Database | Query |
|----------|-------|
| MySQL | `SELECT @@version` หรือ `SELECT version()` |
| PostgreSQL | `SELECT version()` |
| MSSQL | `SELECT @@version` |
| Oracle | `SELECT * FROM v$version` |
| SQLite | `SELECT sqlite_version()` |

### String Concatenation

| Database | Syntax |
|----------|--------|
| MySQL | `CONCAT('a', 'b')` หรือ `'a' 'b'` (space) |
| PostgreSQL | `'a' \|\| 'b'` |
| MSSQL | `'a' + 'b'` |
| Oracle | `'a' \|\| 'b'` หรือ `CONCAT('a', 'b')` |

### Comments

| Database | Single Line | Multi Line |
|----------|-------------|------------|
| MySQL | `-- ` (ต้องมี space) หรือ `#` | `/* */` |
| PostgreSQL | `--` | `/* */` |
| MSSQL | `--` | `/* */` |
| Oracle | `--` | `/* */` |

---

## SQL Query Execution Order

```sql
SELECT category, COUNT(*) as total     -- 5. SELECT (เลือก columns)
FROM products                          -- 1. FROM (เลือก table)
WHERE price > 100                      -- 2. WHERE (filter rows)
GROUP BY category                      -- 3. GROUP BY (จัดกลุ่ม)
HAVING COUNT(*) > 5                    -- 4. HAVING (filter groups)
ORDER BY total DESC                    -- 6. ORDER BY (เรียง)
LIMIT 10;                              -- 7. LIMIT (จำกัดจำนวน)
```

**Execution Order:**
```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

**ทำไมต้องรู้:** เข้าใจว่า inject ที่จุดไหน query จะถูก process ยังไง

---

## Database to Web — เบื้องหลังการทำงาน

### Web Application Architecture

```
┌──────────┐    HTTP     ┌──────────────┐    SQL Query    ┌──────────┐
│  Browser │ ◄────────► │  Web Server  │ ◄─────────────► │ Database │
│ (Client) │  Request/   │ (Backend)    │    Results      │ (MySQL,  │
│          │  Response   │              │                 │ PostgreSQL)
└──────────┘             └──────────────┘                 └──────────┘
     │                         │                               │
     │                         │                               │
  HTML/JS                 PHP/Python/                     Data Storage
  CSS                     Node.js/Java                    Tables/Rows
```

### Request Flow — จาก Click ถึง Database

**Scenario:** User search สินค้า category "Gifts"

#### Step 1: Browser ส่ง HTTP Request
```http
GET /products?category=Gifts HTTP/1.1
Host: shop.example.com
```

#### Step 2: Web Server รับ Request
```python
# Python Flask example
@app.route('/products')
def get_products():
    category = request.args.get('category')  # รับค่า "Gifts"
```

#### Step 3: สร้าง SQL Query
```python
    # ❌ VULNERABLE: String concatenation
    query = "SELECT * FROM products WHERE category = '" + category + "'"

    # Query ที่ได้:
    # SELECT * FROM products WHERE category = 'Gifts'
```

#### Step 4: ส่ง Query ไป Database
```python
    cursor.execute(query)
    results = cursor.fetchall()
```

#### Step 5: Database ประมวลผล
```
Database Engine:
1. Parse SQL syntax
2. ค้นหา table "products"
3. Filter rows ที่ category = 'Gifts'
4. Return matching rows
```

#### Step 6: ส่งผลลัพธ์กลับ Browser
```python
    return render_template('products.html', products=results)
```

### Database Connection — Backend เชื่อมต่อ DB ยังไง

```python
# Python
import psycopg2

conn = psycopg2.connect(
    host="localhost",
    database="shop_db",
    user="app_user",
    password="db_password123",
    port=5432
)
```

```php
// PHP
$conn = new mysqli("localhost", "app_user", "db_password123", "shop_db");
```

```javascript
// Node.js
const pool = new Pool({
    host: 'localhost',
    database: 'shop_db',
    user: 'app_user',
    password: 'db_password123',
    port: 5432
});
```

---

## Vulnerable vs Secure Query Building

### ❌ Vulnerable: String Concatenation

```python
username = request.form['username']  # User input: admin
password = request.form['password']  # User input: ' OR '1'='1

query = "SELECT * FROM users WHERE username='" + username + "' AND password='" + password + "'"

# Result:
# SELECT * FROM users WHERE username='admin' AND password='' OR '1'='1'
#                                                           ↑
#                                              Injected SQL logic!
```

**ปัญหา:** Database ไม่รู้ว่าส่วนไหนคือ **data** ส่วนไหนคือ **code**

### ✅ Secure: Prepared Statements (Parameterized Queries)

```python
# Python (psycopg2)
query = "SELECT * FROM users WHERE username = %s AND password = %s"
cursor.execute(query, (username, password))
```

```php
// PHP (PDO)
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ? AND password = ?");
$stmt->execute([$username, $password]);
```

```javascript
// Node.js (pg)
const query = 'SELECT * FROM users WHERE username = $1 AND password = $2';
const values = [username, password];
await pool.query(query, values);
```

**ทำไมปลอดภัย:** Input ถูก treat เป็น **literal string** เสมอ ไม่สามารถ escape ออกมาเป็น SQL syntax ได้

---

## สรุป SQL ที่จำเป็นสำหรับ SQLi

| Concept | ทำไมต้องรู้ |
|---------|-----------|
| SELECT, WHERE | เข้าใจ query ที่ถูก inject |
| UNION | รวม query เพื่อ extract data |
| ORDER BY | หาจำนวน columns |
| JOIN | เข้าใจ query ที่ซับซ้อน |
| information_schema | enumerate DB structure |
| SUBSTRING | Blind SQLi ทีละตัวอักษร |
| Comments | ตัด query ส่วนที่เหลือ |
| DB-specific syntax | ต้องรู้ว่ากำลัง attack DB อะไร |

---

## Next Steps

- [02-SQL-Injection-Basics.md](./02-SQL-Injection-Basics.md) — Detection & Basic Exploitation
- [03-UNION-Attack.md](./03-UNION-Attack.md) — UNION-based SQLi
- [04-Blind-SQLi.md](./04-Blind-SQLi.md) — Boolean & Time-based Blind
- [05-SQLmap-Usage.md](./05-SQLmap-Usage.md) — Automation with SQLmap
