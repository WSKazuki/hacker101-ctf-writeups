# A Little Something — Hacker101 CTF Writeup

**Platform:** Hacker101  
**Difficulty:** Easy  
**Flags:** 1 / 1 ✅  
**Author:** Hosam A. Ghanima

---

## Overview

A static page with nothing visible — the trick is finding a hidden file that was never linked anywhere.

---

## Solution

**Vulnerability:** Forced Browsing  
**Location:** Static assets directory

### Steps
1. Open the page — nothing interesting in the HTML source
2. Guess common static file names: try `/robots.txt`, `/background.png`, `/favicon.ico`
3. Navigate directly to `/background.png`
4. The flag is embedded inside the image file (visible in the response or metadata)

### Why it works
The file exists on the server but is never linked from any page. There's no access control on it — if you know (or guess) the path, you can reach it directly. This is **forced browsing**: enumerating paths the app didn't intend to expose.

**Key lesson:** Always check static assets. Images, JS files, and other resources can leak sensitive data even when they're not visible in the UI.

**Flag:** `^FLAG^bc67a83f4ef4d4e65f80869c37fc21e1cdaafd06b04ac0f7f64a1b8a4e55b0f2$FLAG$`
