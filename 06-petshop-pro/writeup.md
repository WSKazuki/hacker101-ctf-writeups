# Petshop Pro — Hacker101 CTF Writeup

**Difficulty:** Easy  
**Flags:** 2 / 3 (third flag — admin login — in progress)  
**Category:** Web

---

## Overview

Petshop Pro is a small e-commerce demo where you buy a "Kitten" or "Puppy" photo. Two independent vulnerabilities yield the first two flags:

1. **Client-Side Price Manipulation** — the entire cart (including prices) is a JSON blob in a hidden form field; the server never recalculates prices server-side, so editing that field before submission gets you Flag 1.
2. **Broken Access Control → Stored XSS** — a product edit endpoint (`/edit?id=<id>`) is fully unauthenticated and takes unsanitized input; injecting a stored XSS payload through it exposes Flag 2.

---

## Recon

The app has two products (Kitten, Puppy) and two visible actions per product: `add/<id>` to add to cart and a checkout flow. The DOM and source are the first things to read — they reveal the cart structure immediately. The existence of `/add/<id>` also telegraphs that sibling CRUD endpoints may exist at predictable paths.

---

## Flag 1 — Client-Side Price Trust

### The vulnerability

Viewing `/cart` page source shows the entire cart encoded in a single hidden form field submitted to `/checkout`:

```html
<input type="hidden" name="cart"
  value="[[0, {&quot;name&quot;: &quot;Kitten&quot;, ..., &quot;price&quot;: 8.95}], ...]">
```

The server accepts whatever numeric value arrives in `price` — it never re-fetches the real price from its own catalog.

### Exploit

1. Open DevTools → Elements, find the hidden `cart` input on `/cart`.
2. Right-click → **Edit as HTML**, change `"price": 8.95` to `"price": 0`.
3. Submit the form normally.
4. The confirmation page returns Flag 1 whenever any item price is ≤ 0.

**Bonus:** setting `"price": 1e400` renders `$inf` in the total — the field takes arbitrary numeric input with no bounds check, but no separate flag is awarded for it.

### Fix

Never trust price, quantity, or total values from the client. Recompute them server-side at checkout from a trusted product catalog keyed by product ID only.

---

## Flag 2 — Broken Access Control → Stored XSS

### Discovery

`/add/<id>` suggested a CRUD API. Testing the sibling endpoint `/edit?id=0` directly (no credentials) returned a full product edit form — name, description, price — with no authentication at all, despite `/admin/login` existing elsewhere in the app.

**Unlinked ≠ unreachable.** The UI just didn't show a link.

### Exploit

1. Navigate to `/edit?id=1` — the Puppy product — unauthenticated.
2. Replace the `name` field with a stored XSS payload:
```html
   <img src=x onerror=alert(1)>
```
3. Submit (Save).
4. Reload the app fresh, add Puppy to cart, go to `/cart`.
5. The payload executes in the browser — `alert(1)` fires, confirming the payload is stored server-side and served unsanitized.
6. Flag 2 appears directly on `/cart`:

### Why it matters

- Any visitor who views the cart with that product in it runs the attacker's JavaScript — not just the attacker's own session.
- No auth on `/edit` means any unauthenticated user can modify product data permanently.
- Combined: broken access control + stored XSS = session/cookie theft, defacement, phishing overlays.

### Fix

- Enforce authentication **and** authorization on every state-changing endpoint, not just the ones the UI links to.
- HTML-encode all user-controlled output before rendering it (`name`, `description`).
- Add a strict `Content-Security-Policy` header as defense-in-depth.

---

## Flag 3 — Status: In Progress

An `/admin/login` form exists with a username-enumeration flaw ("Invalid username" vs. a different message for wrong password). SQLi attempts and ~850-entry credential wordlist (default creds + common admin names) came back clean. The official hint references the "entrypoint" — further endpoint discovery rather than credential attacks is the likely path forward.

---

## Key Takeaways

- **DOM edits vs. real requests.** Editing the rendered DOM (e.g., changing text on the homepage) has zero effect — only edits to fields that are actually submitted in a form/request matter. The hidden `cart` field works because it is part of the POST body sent to the server.
- **Guess
 CRUD siblings.** If `/add/<id>` exists, test `/edit`, `/delete`, `/update` at the same path depth before running a wordlist. Pattern recognition beats brute force here.
- **Verify automated results manually.** A concurrent brute-force script produced dozens of false "FOUND" results — the server was returning a 503 error page under load, not a real match. Always confirm with a single clean request before trusting tool output.
