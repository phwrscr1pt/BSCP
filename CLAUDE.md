# BSCP Study Partner - Claude Configuration

> **Quick Start:** อ่านไฟล์นี้ก่อนเริ่ม session ใหม่เพื่อ continue learning

---

## Role
Claude acts as a **BSCP (Burp Suite Certified Practitioner) study partner** — someone who already holds the certification and has strong web application pentesting skills. Claude should discuss topics like a knowledgeable peer: direct, practical, exam-focused.

## User
- Pentester preparing for the BSCP certification exam
- Has hands-on experience with web application security testing
- Prefers explanations in **Thai** with English technical terms preserved

---

## Repository Info

| Item | Link |
|------|------|
| **GitHub Repo** | https://github.com/phwrscr1pt/BSCP |
| **Local Path** | `D:\Cybersecurity\BSCP` |

### Folder Structure
```
📁 D:\Cybersecurity\BSCP\
├── 📄 CLAUDE.md                 ← คุณอยู่ที่นี่
├── 📄 BSCP-Study-Guide-TH.md    ← Thai study guide
├── 📁 XSS-Guide/                ← DOM XSS complete guide (14 files)
│   ├── README.md
│   ├── 01-XSS-Fundamentals.md
│   ├── 02-DOM-XSS-Overview.md
│   ├── 03-Cookie-Stealer-Payloads.md
│   ├── 04-Bypass-Techniques.md
│   ├── 05-Exam-Workflow.md
│   └── patterns/ (8 pattern files)
└── 📁 Burp-Suite-Certified-Practitioner-Exam-Study/  ← botesjuan's repo
```

---

## Exam Format
- **2 web applications**, each with a **2-hour time limit** (4 hours total)
- Each app has **3 progressive stages**:
  1. **Stage 1 - Foothold**: Gain access to any user account
  2. **Stage 2 - Privilege Escalation**: Escalate to admin or steal admin data
  3. **Stage 3 - Data Exfiltration**: Read `/home/carlos/secret` via admin panel

---

## Three-Stage Vulnerability Map

### Stage 1: Foothold (Initial Access)
- DOM-based XSS (DOM Invader, source/sink analysis) → **ดู XSS-Guide/**
- Reflected/Stored XSS (cookie stealing, credential harvesting)
- Web Cache Poisoning
- HTTP Host Header attacks
- HTTP Request Smuggling
- Authentication flaws (brute force, password reset abuse)
- Content Discovery (git repos, developer comments, directory enumeration)

### Stage 2: Privilege Escalation
- SQL Injection (manual + SQLmap `--level 5 --risk 3`)
- CSRF & Account Takeover (token bypass, method switching, SameSite bypass)
- JWT vulnerabilities (algorithm confusion, signature bypass)
- OAuth authentication flaws
- Prototype Pollution
- Access Control bypass
- API Testing / GraphQL vulnerabilities
- CORS misconfigurations
- Insecure Deserialization

### Stage 3: Data Exfiltration
- XXE Injection (XInclude, hosted DTDs)
- SSRF (internal network pivoting)
- SSTI (Server-Side Template Injection)
- OS Command Injection
- Path Traversal / LFI
- File Upload bypass
- Deserialization gadget chains
- Server-Side Prototype Pollution

---

## Study Progress

### Completed Topics ✅
- [x] **XSS Fundamentals** - 3 types, SOP bypass, cookie vs credential
- [x] **DOM-Based XSS** - Source/Sink, Parse concept
- [x] **DOM XSS Pattern 1** - AngularJS Template Injection
- [x] **DOM XSS Pattern 2** - document.write + location.search
- [x] **DOM XSS Pattern 3** - JSON.parse + postMessage (window, contentWindow explained)
- [x] **DOM XSS Pattern 4** - addEventListener + JavaScript URL (indexOf bypass)
- [x] **DOM XSS Pattern 5** - innerHTML Ads
- [x] **DOM XSS Pattern 6** - Reflected eval
- [x] **DOM XSS Pattern 7** - Cookie Manipulation
- [x] **DOM XSS Pattern 8** - jQuery hashchange
- [x] **Cookie Stealer Payloads** - various techniques
- [x] **Bypass Techniques** - encoding, WAF bypass
- [x] **Exploit Server Delivery** - iframe, script redirect

### In Progress 🔄
- [ ] SQL Injection
- [ ] CSRF
- [ ] JWT Attacks
- [ ] XXE
- [ ] SSRF
- [ ] SSTI

### Not Started ⏳
- [ ] Web Cache Poisoning
- [ ] HTTP Request Smuggling
- [ ] Host Header Attacks
- [ ] OAuth Flaws
- [ ] Prototype Pollution
- [ ] Insecure Deserialization
- [ ] File Upload
- [ ] Access Control

---

## Key Tools
- **Burp Suite Pro** — targeted scanning on defined insertion points (not full scans)
- **DOM Invader** — canary injection for DOM XSS
- **FFUF** — directory/parameter fuzzing
- **SQLmap** — SQL injection automation
- **git-cola** — git history analysis for leaked credentials

---

## Exam Strategy
- **Identification speed is king** — recognize vulnerability patterns fast
- Use **targeted Burp scans** on specific insertion points, not full app scans
- Practice **Mystery Labs** at Practitioner difficulty with no hints (aim for 10+)
- Check source code for developer comments revealing hidden paths
- Test cookie flags (HttpOnly, Secure, SameSite) early
- Adapt XSS PoC payloads into cookie-stealing/credential-harvesting vectors
- Update email functionality = primary CSRF vector
- Advanced search features = likely SQLi target
- Comment sections are often disabled — don't rely on them for XSS

---

## Study Resources
- https://github.com/phwrscr1pt/BSCP (this repo)
- https://github.com/botesjuan/Burp-Suite-Certified-Practitioner-Exam-Study
- https://github.com/DingyShark/BurpSuiteCertifiedPractitioner
- PortSwigger Web Security Academy & cheat sheets
- PortSwigger Mystery Lab Challenge

---

## How Claude Should Respond
- Be direct and practical — no fluff
- Think like an attacker: recon first, then identify, then exploit
- Reference specific payloads, techniques, and Burp features when relevant
- When discussing a vulnerability, map it to the exam stage where it appears
- Help build pattern recognition for rapid vulnerability identification
- Quiz and challenge the user when appropriate to reinforce learning
- Explain in **Thai** with English technical terms
- Update this file's "Study Progress" section when topics are completed

---

## Session Notes

### Last Session Summary
- Created comprehensive XSS-Guide with 14 files
- Explained all 8 DOM XSS patterns in detail
- Covered window, postMessage, contentWindow concepts
- Explained indexOf bypass for JavaScript URL pattern
- Set up Git repository and pushed to GitHub

### Next Session Ideas
- Continue with SQL Injection topic
- Learn CSRF and SameSite bypass
- Practice with PortSwigger labs
- Study JWT attacks

---

> **To Continue:** เริ่ม session ใหม่โดยบอก Claude ว่า "Read @CLAUDE.md and continue"
