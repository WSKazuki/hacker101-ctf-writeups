# Micro-CMS v1 — Hacker101 CTF Writeup

**Platform:** Hacker101  
**Difficulty:** Easy  
**Flags:** 4 / 4 ✅  
**Date:** September 2026  
**Author:** Hosam A. Ghanima

---

## Overview

A simple CMS application that lets users create and edit markdown pages. Four vulnerabilities to find — two XSS, one SQLi, one unauthorized access.

---

## Recon

The app has three pages listed on the home page:
- Testing
- Markdown Test
- *(empty bullet — hidden page with no title)*

Key endpoints discovered:

---

## Flag 0 — Stored XSS in Title Field

**Vulnerability:** Stored XSS  
**Location:** Title input on the Create Page form

### Steps
1. Go to Create Page
2. In the **Title** field, enter:
```html
   <script>alert(1)</script>
```
3. Save and view the page — the script executes

### Why it works
The title field is rendered without sanitization on the page view. The `<script>` tag executes directly in the browser.

**Key lesson:** Always test title/name/label inputs — they often bypass sanitization applied to body content.

**Flag:** `^FLAG^bc332c8ff2f53e145b9bec171d985227031f57523d09425dec24ce2ba0f49296$FLAG$`

---

## Flag 1 — Stored XSS in Markdown Body (Event Handler Bypass)

**Vulnerability:** Stored XSS via HTML event handler  
**Location:** Body/content textarea (markdown field)

### Steps
1. Edit any page
2. The app says *"Markdown is supported, but scripts are not"* — `<script>` is filtered
3. Bypass using an HTML event handler in the **body** field:
```html
   <img src=x onerror=alert(document.cookie)>
```
4. Save and view the page — the `onerror` handler fires because `src=x` fails to load

### Why it works
The app blocks `<script>` tags but does not sanitize HTML event attributes like `onerror`. The markdown renderer passes raw HTML through, so event handlers execute freely.

**Key lesson:** Blocking `<script>` is not XSS-safe. Always test `onerror`, `onload`, `onmouseover`, etc.

**Flag:** `^FLAG^77497547d14b56bc8fab33c696bf74bf31bfd6ef61e46665366f33b58e7ab13f$FLAG$`

---

## Flag 2 — SQL Injection on Edit Endpoint

**Vulnerability:** SQL Injection (error-based)  
**Location:** Page ID parameter in `/page/edit/<id>`

### Steps
1. Notice the URL structure: `/page/edit/1`
2. Append a single quote to break the SQL query: `/page/edit/1'`
3. Add `--` to comment out the rest: `/page/edit/1'--`
4. Flag is returned in the error response

### Why it works
The page ID is passed directly into a SQL query without parameterization. The `'` breaks the query syntax; `--` comments out everything after, triggering an error that leaks the flag.

**Key lesson:** Always test edit/update endpoints for SQLi — not just search/view endpoints.

**Flag:** `^FLAG^7ea18ad277447ecb05c03a76d615c607fe0a098c4ad2cc98563efd875d541d9b$FLAG$`

---

## Flag 3 — Unauthorized Access to Forbidden Page

**Vulnerability:** Broken Access Control  
**Location:** `/page/5` (hidden page, no title)

### Steps
1. On the home page, the 3rd bullet point is **empty** (page with no title)
2. Clicking it leads to `/page/5` → **403 Forbidden**
3. Try accessing the edit endpoint directly: `/page/edit/5`
4. The server returns the flag — access control only enforced on the view route

### Why it works
The application checks permissions on `/page/<id>` but fails to apply the same check on `/page/edit/<id>`. Inconsistent access control — very common in real web apps.

**Key lesson:** Always test edit/delete/admin endpoints separately from view endpoints. Access control logic is often missed on some routes.

**Flag:** `^FLAG^2799a5a2735cce8fd0c348d1fd1a15f28a459a62d0e1b05d151d728cc3cc31b8$FLAG$`

---

## Summary

| Flag | Vulnerability | Location |
|------|--------------|----------|
| 0 | Stored XSS | Title field — `<script>` tag |
| 1 | Stored XSS (filter bypass) | Body field — `<img onerror>` |
| 2 | SQL Injection | `/page/edit/1'--` |
| 3 | Broken Access Control | `/page/edit/5` (forbidden page) |

## Tools Used
- Browser (manual testing)
- Burp Suite (request inspection)
