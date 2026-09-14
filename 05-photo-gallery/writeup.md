# Photo Gallery — Hacker101 CTF Writeup

**Difficulty:** Easy  
**Flags:** 3  
**Category:** Web

---

## Overview

A Flask/Python 2.7 photo gallery app. Two vulnerabilities chain together to get all three flags in a single shot:

1. **SQL Injection → LFI** — read `main.py` via UNION SELECT, revealing Flag 1 and the full source code
2. **Stacked Queries → Command Injection** — one UPDATE payload triggers a shell injection that dumps the FLAGS environment variable, revealing all three flags at once

---

## Recon

The `/fetch?id=X` endpoint serves photo images. Testing edge cases:

- `/fetch?id=1'` → 500 (exception) — string interpolation in SQL, not parameterized
- `/fetch?id=99` → 404 — `abort(404)` when no DB row found
- `/fetch?id=3` → 500 — row exists but the file doesn't ("Invisible" photo is a hint)

**HTTP status oracle:** `200` = row found + file readable, `404` = no row, `500` = row found but file missing/error.

---

## Step 1 — Read the Source (Flag 1)

The `/fetch` route uses string interpolation:

```python
cur.execute('SELECT filename FROM photos WHERE id=%s' % request.args['id'])
return file('./%s' % cur.fetchone()[0].replace('..', ''), 'rb').read()
```

The filename returned by the query is passed straight to `file()`. Inject a UNION SELECT to control the filename — use a non-existent photo ID so the real query returns 0 rows, then UNION in our target file:

**Important:** open this in `curl` or view-source, not the browser — the browser renders the HTML inside the template strings and hides the Python code behind it.

The app reads `./main.py` and returns the full Flask source. The code contains a hardcoded flag in a comment:

```python
# It's dangerous to go alone, take this:
# ^FLAG^3897a947bc226da25420c8e18b37c061c7da78eb5c8d6cd60ae920dd961d4364$FLAG$
```

Reading the source also reveals the two other vulnerabilities we'll exploit next.

**Flag 1:** `^FLAG^3897a947bc226da25420c8e18b37c061c7da78eb5c8d6cd60ae920dd961d4364$FLAG$`

---

## Step 2 — Understand the Command Injection (from the source)

The homepage code in `main.py`:

```python
cur.execute('SELECT id, title, filename FROM photos WHERE parent=%s LIMIT 3', (id, ))
fns = []
for pid, ptitle, pfn in cur.fetchall():
    fns.append(pfn)
rep += 'Space used: ' + subprocess.check_output(
    'du -ch %s || exit 0' % ' '.join('files/' + fn for fn in fns),
    shell=True,
    stderr=subprocess.STDOUT
).strip().rsplit('\n', 1)[-1]
```

Photo filenames from the DB are concatenated into a shell command with `shell=True`. If a filename contains `;`, the shell splits it into separate commands. The **last line** of the subprocess output is shown in "Space used."

The homepage query is parameterized (no direct injection), but the `/fetch` SQLi can **modify the DB** via stacked queries to plant a malicious filename.

---

## Step 3 — Stacked Query UPDATE + Commit (Flags 2 and 3)

### The payload

`id=4` → `SELECT filename FROM photos WHERE id=4` returns 0 rows. MySQL still processes the full multi-statement string, running the UPDATE and COMMIT before Python reads the result.

**Why `;commit;` is essential:** `MySQLdb.connect()` has autocommit OFF by default. The homepage creates a fresh DB connection via `getDb()` on every request. Without an explicit COMMIT, an UPDATE in one connection's transaction is invisible to all other connections. The `;commit;` flushes the change so the homepage immediately sees the new filename.

### What happens after

Photo 3's filename in the DB is now `; echo $(printenv)`.

When the homepage loads, it builds:

```bash
du -ch files/adorable.jpg files/purrfect.jpg files/; echo $(printenv) || exit 0
```

Shell parsing splits at `;`:
1. `du -ch files/adorable.jpg files/purrfect.jpg files/` — runs normally
2. `echo $(printenv)` — dumps all environment variables as the last line of output

`rsplit('\n', 1)[-1]` captures that last line. The CTF platform injects all challenge flags into the container environment under `FLAGS`:

All three flags appear at once in the "Space used" field on the homepage.

---

## All Flags

| # | Flag |
|---|------|
| 1 | `^FLAG^3897a947bc226da25420c8e18b37c061c7da78eb5c8d6cd60ae920dd961d4364$FLAG$` |
| 2 | `^FLAG^91cf5255add564e193c4c1cdb884771b104697b1da10f884519d3a8315f61089$FLAG$` |
| 3 | `^FLAG^887023e00cf23b4665f8e70d8ae809b6d14bf3bc32f7dfbb4d99b17855eb9edc$FLAG$` |

---

## Key Takeaways

- **Read the source first.** One UNION SELECT LFI to `main.py` hands you every other vulnerability on a plate — no guessing required.
- **Use curl, not the browser.** HTML inside source files gets parsed; the Python code disappears behind rendered markup.
- **Always include `;commit;` with stacked write queries.** MySQLdb's default autocommit=OFF means UPDATE changes are invisible to other connections until committed. Missing this makes it look like stacked queries don't work at all.
- **`$(printenv)` > guessing file paths.** CTF containers store flags in env variables. One echo dumps everything.
