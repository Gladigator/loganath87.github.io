---
title: "Bug Bounty Web Application Testing Checklist - Complete Methodology"
date: 2025-12-27 12:00:00 +1100
categories: [Cybersecurity, Bug Bounty]
tags: [bug-bounty, web-security, penetration-testing, owasp, checklist, methodology]
pin: false
image:
  path: /assets/img/bug-bounty-checklist.png
  alt: Bug Bounty Web Application Testing Checklist
---

A comprehensive, systematic methodology for bug bounty hunters and web application penetration testers. This checklist ensures you never miss a vulnerability check during your security assessments.

## 📋 Overview

This checklist provides a complete, step-by-step methodology covering:

- ✅ **4 Comprehensive Testing Phases**
- ✅ **150+ Vulnerability Checks**
- ✅ **All OWASP Top 10 Vulnerabilities**
- ✅ **Essential Tools Reference**
- ✅ **Quick Payloads Library**

Perfect for bug bounty hunters, pentesters and security researchers who want to ensure thorough coverage of all vulnerability types.

---

## 📥 Downloads

- [📄 DOCX Version](/downloads/Bug_Bounty_Checklist.docx) - Print and check off items
- [📝 Markdown Version](/downloads/Bug_Bounty_Checklist.md) - Use in terminal or notes
- [💻 Interactive HTML](/checklist.html) - Web-based version

---

## 🔍 Testing Methodology

### Phase 1: Reconnaissance & Information Gathering

Build a complete picture of your target before diving into testing.

#### 1.1 Domain & Subdomain Enumeration
- [ ] Find all subdomains using subfinder/amass
- [ ] Check DNS records (A, AAAA, MX, TXT, CNAME)
- [ ] Search for domains in crt.sh (certificate transparency)
- [ ] Use Google dorking for subdomains
- [ ] Check SecurityTrails/VirusTotal for historical data

#### 1.2 Technology Detection
- [ ] Identify web server (Apache, Nginx, IIS)
- [ ] Detect CMS (WordPress, Drupal, Joomla)
- [ ] Find JavaScript frameworks (React, Angular, Vue)
- [ ] Check for CDN usage (Cloudflare, Akamai)
- [ ] Identify WAF/security solutions
- [ ] Note programming language (PHP, Python, Java, Node.js)
- [ ] Check HTTP headers for technology clues

#### 1.3 Port Scanning & Service Discovery
- [ ] Scan common ports (80, 443, 8080, 8443, 3000, 8000)
- [ ] Full port scan (all 65535 ports)
- [ ] Service version detection
- [ ] Check for open admin panels (3306, 27017, 5432, 6379)
- [ ] Look for exposed services (SSH, FTP, SMB)

---

### Phase 2: Application Mapping & Analysis

Map the entire application surface and understand its behavior.

#### 2.1 Directory & File Discovery
- [ ] Run gobuster/ffuf with common wordlist
- [ ] Try extensions: .php, .asp, .aspx, .jsp, .html, .js
- [ ] Look for backup files: .bak, .old, .backup, ~
- [ ] Find config files: web.config, .env, config.php
- [ ] Check for .git, .svn, .DS_Store exposure
- [ ] Look for admin panels: /admin, /administrator, /wp-admin
- [ ] Find API endpoints: /api, /v1, /graphql

#### 2.2 Spider & Crawl Application
- [ ] Use Burp Suite spider/crawler
- [ ] Map all pages and functionalities
- [ ] Identify all input points (forms, parameters)
- [ ] Find all cookies and headers
- [ ] List all API endpoints
- [ ] Discover hidden parameters with Arjun/ParamSpider

#### 2.3 Analyze Application Behavior
- [ ] Understand authentication flow
- [ ] Map user roles and privileges
- [ ] Identify session management mechanism
- [ ] Note how errors are handled
- [ ] Check for client-side security controls
- [ ] Analyze JavaScript files for hardcoded secrets

---

### Phase 3: Vulnerability Testing

Systematic testing covering all major vulnerability categories.

#### 3.1 Injection Vulnerabilities

##### SQL Injection (SQLi)
- [ ] Test all input fields with: `' " \ ; -- # /*`
- [ ] Try UNION-based SQLi
- [ ] Test error-based SQLi
- [ ] Try boolean-based blind SQLi
- [ ] Test time-based blind SQLi
- [ ] Use SQLMap for automated testing
- [ ] Check POST, GET, Cookie and Header parameters

##### Cross-Site Scripting (XSS)
- [ ] Test reflected XSS: `<script>alert(1)</script>`
- [ ] Test stored/persistent XSS
- [ ] Test DOM-based XSS
- [ ] Try HTML injection
- [ ] Test different contexts (attributes, JavaScript, CSS)
- [ ] Bypass filters: `<img src=x onerror=alert(1)>`
- [ ] Try polyglot payloads
- [ ] Test in ALL input fields and parameters

##### Command Injection
- [ ] Test with: `; | || & && \` $ ( )`
- [ ] Try: `;ls;`  `|whoami|`  `` `ping -c 1 attacker.com` ``
- [ ] Test file upload functionality
- [ ] Check for code execution in file operations

##### XML/XXE Injection
- [ ] Test XML parsers with XXE payloads
- [ ] Try file disclosure via XXE
- [ ] Test SSRF via XXE
- [ ] Check SOAP endpoints

##### Template Injection
- [ ] Test Server-Side Template Injection (SSTI)
- [ ] Try: `{{7*7}}` `{{config}}` `${7*7}`
- [ ] Check for code execution via templates

#### 3.2 Authentication & Session Vulnerabilities
- [ ] Test for default credentials (admin/admin, root/root)
- [ ] Check for username enumeration
- [ ] Test password reset functionality
- [ ] Try brute force attacks (rate limiting?)
- [ ] Test for 2FA bypass
- [ ] Check session fixation
- [ ] Test session timeout
- [ ] Try cookie manipulation
- [ ] Test 'Remember Me' functionality
- [ ] Check for password in URL/logs
- [ ] Test CAPTCHA bypass
- [ ] Check OAuth/SSO implementation

#### 3.3 Authorization & Access Control
- [ ] Test IDOR (change IDs in requests)
- [ ] Test horizontal privilege escalation
- [ ] Test vertical privilege escalation
- [ ] Try accessing admin functions as regular user
- [ ] Test direct object references
- [ ] Check forced browsing to restricted pages
- [ ] Test parameter manipulation for access
- [ ] Try HTTP verb tampering (GET→POST)
- [ ] Check for missing function-level access control

#### 3.4 Business Logic Vulnerabilities
- [ ] Test negative numbers in price fields
- [ ] Try race conditions (simultaneous requests)
- [ ] Test workflow bypass
- [ ] Check for unlimited discount/coupon usage
- [ ] Test payment manipulation
- [ ] Try bulk operations abuse
- [ ] Test for logic flaws in multi-step processes
- [ ] Check for business rule violations

#### 3.5 CSRF (Cross-Site Request Forgery)
- [ ] Check if CSRF tokens are implemented
- [ ] Try removing CSRF token
- [ ] Try using another user's token
- [ ] Try changing POST to GET
- [ ] Test CSRF on sensitive actions (password change, etc.)
- [ ] Check if tokens are validated
- [ ] Test same-site cookie attributes

#### 3.6 SSRF (Server-Side Request Forgery)
- [ ] Test URL parameters with internal IPs
- [ ] Try: `http://127.0.0.1`, `http://localhost`
- [ ] Test cloud metadata endpoints (`169.254.169.254`)
- [ ] Try bypasses: `http://127.1`, `http://0`, `http://[::1]`
- [ ] Test `file://` protocol
- [ ] Check for port scanning via SSRF

#### 3.7 File Upload Vulnerabilities
- [ ] Try uploading web shells (.php, .asp, .jsp)
- [ ] Test double extension bypass (.php.jpg)
- [ ] Try null byte injection (shell.php%00.jpg)
- [ ] Upload files with special names (../../../shell.php)
- [ ] Test content-type bypass
- [ ] Try polyglot files (valid image + malicious code)
- [ ] Check for path traversal in upload
- [ ] Test file size limits and DoS
- [ ] Upload SVG with embedded JavaScript

#### 3.8 Information Disclosure
- [ ] Check robots.txt and sitemap.xml
- [ ] Look for exposed .git, .svn, .env files
- [ ] Check source code comments
- [ ] Find sensitive data in JavaScript files
- [ ] Test error messages (verbose errors?)
- [ ] Check HTTP headers (Server, X-Powered-By)
- [ ] Look for backup files
- [ ] Test directory listing
- [ ] Check for exposed admin panels
- [ ] Search for API keys in source

#### 3.9 Security Misconfiguration
- [ ] Check for default configurations
- [ ] Test CORS misconfiguration
- [ ] Check HTTP security headers (CSP, X-Frame-Options, etc.)
- [ ] Test TLS/SSL configuration
- [ ] Check for unnecessary HTTP methods (PUT, DELETE)
- [ ] Look for exposed admin interfaces
- [ ] Test for open redirects
- [ ] Check cookie security (HttpOnly, Secure flags)

#### 3.10 API Testing
- [ ] Find all API endpoints
- [ ] Test for broken authentication
- [ ] Try excessive data exposure
- [ ] Test rate limiting
- [ ] Check for BOLA (Broken Object Level Authorization)
- [ ] Test mass assignment
- [ ] Try security misconfiguration
- [ ] Check for injection in API parameters
- [ ] Test GraphQL introspection
- [ ] Look for API versioning issues

#### 3.11 Client-Side Vulnerabilities
- [ ] Test clickjacking (X-Frame-Options?)
- [ ] Check for DOM-based XSS
- [ ] Test HTML5 security issues
- [ ] Check WebSocket security
- [ ] Test postMessage vulnerabilities
- [ ] Look for sensitive data in localStorage/sessionStorage
- [ ] Check for JavaScript prototype pollution

#### 3.12 Advanced Attacks
- [ ] Test for deserialization vulnerabilities
- [ ] Try HTTP request smuggling
- [ ] Test for Host Header injection
- [ ] Check for OAuth vulnerabilities
- [ ] Test JWT security
- [ ] Look for NoSQL injection
- [ ] Test for XML bomb (billion laughs)
- [ ] Check for LDAP injection
- [ ] Test for cache poisoning

---

### Phase 4: Documentation & Reporting

Create professional, actionable reports.

#### 4.1 Document Your Findings
- [ ] Vulnerability title and severity
- [ ] Detailed description
- [ ] Steps to reproduce
- [ ] Proof of concept (PoC)
- [ ] Screenshots/videos
- [ ] Impact assessment
- [ ] Remediation recommendations
- [ ] CVSS score (if applicable)

#### 4.2 Report Quality
- [ ] Clear and concise writing
- [ ] Proper technical detail
- [ ] Reproducible steps
- [ ] No duplicate reports
- [ ] Follow program's guidelines
- [ ] Include all relevant information
- [ ] Professional tone

---

## 🛠️ Essential Tools Reference

### Reconnaissance
- subfinder, amass, assetfinder, crt.sh

### Directory Discovery
- gobuster, ffuf, dirsearch, feroxbuster

### Vulnerability Scanning
- nuclei, nikto, nmap, SQLMap

### Web Proxy
- Burp Suite Professional/Community

### Browser Extensions
- Wappalyzer, Cookie-Editor, FoxyProxy

### Parameter Discovery
- Arjun, ParamSpider

### Subdomain Takeover
- subjack, SubOver

### Screenshots
- EyeWitness, Aquatone

### Content Discovery
- waybackurls, gau

### JavaScript Analysis
- JSFinder, LinkFinder

---

## 💉 Quick Payloads Reference

### XSS (Cross-Site Scripting)
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

### SQL Injection
```sql
' OR '1'='1' -- 
' UNION SELECT NULL,NULL,NULL-- 
' AND 1=2 UNION SELECT NULL,version(),NULL-- 
```

### Command Injection
```bash
; whoami ;
| ls -la |
`ping -c 1 attacker.com`
```

### SSRF
```
http://127.0.0.1
http://169.254.169.254/latest/meta-data/
http://localhost
```

### Path Traversal
```
../../etc/passwd
....//....//....//etc/passwd
..%252f..%252f..%252fetc/passwd
```

---

## ⚠️ Important Reminders

> **This checklist is for authorized testing only.**
{: .prompt-danger }

**Always:**
- ✓ Get proper authorization before testing
- ✓ Follow the bug bounty program's scope
- ✓ Practice responsible disclosure
- ✓ Respect privacy and data protection laws

**Never:**
- ✗ Test systems without permission
- ✗ Cause damage or disruption
- ✗ Access or exfiltrate sensitive data
- ✗ Violate laws or regulations

---

## 💡 How to Use This Checklist

1. **For Each Target:**
   - Create a copy of the checklist
   - Work through each phase systematically
   - Document findings as you discover them

2. **Track Progress:**
   ```
   Target: example.com
   [✓] Phase 1: Recon - Complete
   [✓] Phase 2: Mapping - Complete  
   [→] Phase 3: Testing - In Progress
   [ ] Phase 4: Reporting - Not Started
   ```

3. **Document Everything:**
   - Take screenshots
   - Save successful payloads
   - Note interesting findings

---

## 📚 Additional Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [HackerOne Disclosure Guidelines](https://www.hackerone.com/disclosure-guidelines)

### Bug Bounty Platforms
- [HackerOne](https://www.hackerone.com/) - Leading bug bounty platform
- [Bugcrowd](https://www.bugcrowd.com/) - Crowdsourced security testing
- [Intigriti](https://www.intigriti.com/) - European bug bounty platform
- [YesWeHack](https://www.yeswehack.com/) - Global bug bounty platform
- [Synack](https://www.synack.com/) - Vetted researcher platform

---

## 🤝 Contributing

Found a vulnerability type not covered? Have suggestions?  
[Open an issue](https://github.com/Gladigator/gladigator.github.io/issues) or submit a pull request!

---

**Happy Hunting! 🎯**

Save this checklist and use it for every target to ensure comprehensive coverage!
