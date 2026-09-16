# Hacker101 CTF Writeups

**Author:** Hosam A. Ghanima  
**Progress:** 4 complete · 1 in progress

| # | Challenge | Difficulty | Flags | Status |
|---|-----------|------------|-------|--------|
| 01 | A Little Something to Get You Started | Easy | 1/1 | ✅ Done |
| 02 | Micro-CMS v1 | Easy | 4/4 | ✅ Done |
| 03 | Micro-CMS v2 | Moderate | 3/3 | ✅ Done |
| 04 | Photo Gallery | Easy | 3/3 | ✅ Done |
| 05 | Petshop Pro | Easy | 2/3 | 🟡 In Progress |

## Techniques Covered

- Stored XSS (script tag, img onerror, event handler bypass)
- SQL Injection (UNION SELECT, blind boolean, credential extraction)
- SQL Injection → LFI (UNION SELECT to control a filename read by the server, exposing source code)
- Stacked Queries + Explicit COMMIT (MySQLdb autocommit=OFF; UPDATE changes invisible to other connections without `;commit;`)
- Command Injection (shell=True + string interpolation in subprocess calls; `$(printenv)` to dump environment flags)
- Broken Access Control (unlinked admin endpoints reachable without authentication)
- Improper HTTP Method Handling (POST bypasses auth on GET-protected routes)
- Client-Side Price Manipulation (hidden form fields trusted by server; business logic bypass via DOM editing)
