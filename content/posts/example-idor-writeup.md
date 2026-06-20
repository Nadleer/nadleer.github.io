---
title: "Example: IDOR in the Invoices API (template)"
date: 2026-06-19T00:00:00Z
draft: true
tags: ["idor", "bola", "api", "writeup"]
categories: ["findings"]
summary: "A reusable write-up template — copy this file to start a new finding post."
cover:
  image: ""        # optional: put an image in static/images and reference /images/foo.png
  alt: ""
  caption: ""
---

> **This is a draft template.** It won't publish until you set `draft: false`.
> Copy it for every new finding: `cp content/posts/example-idor-writeup.md content/posts/my-new-finding.md`

## TL;DR

One or two sentences: what the bug was, on what asset, and the impact. Example —
*An authenticated user could read any other tenant's invoices by incrementing the `invoice_id` path parameter — full cross-tenant financial data exposure.*

## Target & scope

- **Program:** (public name or "private program")
- **Asset:** `api.example.com`
- **Auth required:** authenticated, lowest-privilege role
- **Vuln class:** IDOR / BOLA

## Summary

Describe the endpoint and the trust assumption it broke. Keep it factual.

## Steps to reproduce

1. Authenticate as User A and capture the request to your own invoice:

   ```http
   GET /v2/invoices/10472 HTTP/2
   Host: api.example.com
   Authorization: Bearer <userA_token>
   ```

2. Change `10472` to an ID belonging to another tenant:

   ```http
   GET /v2/invoices/10001 HTTP/2
   Host: api.example.com
   Authorization: Bearer <userA_token>
   ```

3. Observe the response returns User B's invoice — no authorization check on object ownership.

   ```json
   { "id": 10001, "tenant": "acme-corp", "total": 4820.00, "pdf_url": "..." }
   ```

## Impact

Spell out concrete harm: what data, whose, how much, what an attacker does with it. Tie it to the program's risk.

## Proof

Screenshots / response captures go here. Put images in `static/images/` and reference them as `![](/images/finding-1.png)`.

## Remediation

Enforce object-level authorization: verify the authenticated principal owns (or has a grant to) the requested object before returning it.

## Timeline

- **2026-06-10** — Reported
- **2026-06-14** — Triaged
- **2026-06-18** — Fixed
- **2026-06-19** — Disclosure authorized
