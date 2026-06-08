# CodePath AI301 — Open Source Contribution Journal

**Program:** CodePath AI Open Source Capstone (AI301)  
**Contributor:** Christian Perez  
**Portfolio:** [p2rez.github.io](https://p2rez.github.io)  
**GitHub:** [@p2rez](https://github.com/p2rez)

---

## Contribution Overview

| Field | Details |
|---|---|
| **Project** | [medusajs/medusa](https://github.com/medusajs/medusa) |
| **Issue** | [#14903 — Stale cache served for list responses with caching feature flag enabled](https://github.com/medusajs/medusa/issues/14903) |
| **Issue Type** | Bug |
| **Affected File** | `packages/modules/caching/src/utils/parser.ts` |
| **Status** | 🟡 Phase I — Issue Selected |

---

## What is Medusa?

Medusa is an open-source composable commerce platform built with Node.js. It provides the backend infrastructure for building e-commerce applications — handling products, orders, customers, payments, and more. It's modular by design, meaning features like caching are opt-in via feature flags.

---

## The Bug

When the `caching` feature flag is enabled, list endpoints like `GET /store/products` can return **stale data** after an entity is updated.

**Example scenario:**
1. A product is created in **draft** status
2. `GET /store/products` is called — the response is cached, draft product is excluded
3. The product is **published** via the admin API
4. `GET /store/products` is called again
5. ❌ The cached (stale) response is returned — the published product is missing

The cache doesn't refresh until the TTL expires, meaning users can be served incorrect data for an unknown window of time.

---

## Root Cause

The bug lives in `buildAffectedCacheKeys` inside `parser.ts`. The function decides which cache keys to invalidate when an event fires. Currently, it only triggers list-level invalidation (`Entity:list:*`) for `created` and `deleted` operations:

```ts
if (entity.isInArray || ["created", "deleted"].includes(operation)) {
  keys.add(`${entity.type}:list:*`)
}
```

When a `product.updated` event fires, the invalidation only produces `Product:prod_xyz` — a single entity key. Since the product was *filtered out* of the cached list (it was in draft), no cached list entry has that tag. The stale list response is never cleared.

This affects every entity type where cached list queries use filters — not just products.

---

## Proposed Fix

Extend the condition so that list caches are also invalidated on `updated` operations:

```ts
if (entity.isInArray || ["created", "deleted", "updated"].includes(operation)) {
  keys.add(`${entity.type}:list:*`)
}
```

This ensures any mutation that could change whether an entity matches a filter will properly bust the list cache. Individual entity caches (`Product:prod_123`) are unaffected.

---

## Phase Progress

### ✅ Phase I — Issue Selection (Week 1)
- [x] Identified and read through issue #14903
- [x] Left a comment on the issue to claim it
- [x] Created this Contribution README
- [ ] Fork the medusajs/medusa repository
- [ ] Set up GitHub for contribution

### 🔲 Phase II — Reproduce and Plan (Week 2)
- [ ] Stand up local Medusa dev environment
- [ ] Reproduce the stale cache bug locally
- [ ] Write detailed implementation plan

### 🔲 Phase III — Build (Weeks 3+)
- [ ] Implement the fix in `parser.ts`
- [ ] Write or update relevant tests
- [ ] Push changes to fork regularly

### 🔲 Phase IV — Submit and Iterate (Weeks 4+)
- [ ] Open pull request against medusajs/medusa
- [ ] Respond to maintainer feedback
- [ ] Push revisions as needed

---

## Journal

### Week 1
Selected issue #14903 — a caching bug in Medusa's list invalidation logic. The root cause is already well-documented in the issue by the reporter, which made it a good candidate: the affected file is pinpointed, the fix is scoped, and it's reproducible on any Medusa instance with caching enabled. Left a comment on the issue and set up this journal. Next step is getting the local dev environment running.

---

*Last updated: Week 1 — Phase I*
