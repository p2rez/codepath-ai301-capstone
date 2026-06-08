# Contribution 1: Stale cache served for list responses with caching feature flag enabled

**Contribution Number:** 1  
**Student:** Christian Perez  
**Issue:** [medusajs/medusa #14903](https://github.com/medusajs/medusa/issues/14903)  
**Status:** Phase I — In Progress

---

## Why I Chose This Issue

Caching bugs are tricky because nothing breaks loudly — you just get wrong data and no clue why. That's exactly what's happening here, and I wanted to dig into something that actually requires understanding how the system works rather than just patching an obvious error.

I'm also trying to get more comfortable reading production codebases I didn't write. This issue already has the root cause pinpointed, so instead of spending all my time hunting for the bug, I can focus on understanding the code structure, writing a proper fix, and learning how the contribution process works in a real project. Good starting point for a first PR.

---

## Understanding the Issue

### Problem Description

When caching is turned on in Medusa, list endpoints like `GET /store/products` can return stale data after an entity gets updated. The cache never gets cleared on updates — so if something changes in a way that affects whether it shows up in a filtered list, users just keep getting the old response until the TTL runs out.

### Expected Behavior

Publish a product → it shows up in `GET /store/products` immediately.

### Current Behavior

Publish a product → the cached response from before the update is still returned. The product is missing until the cache expires on its own.

### Affected Components

- **File:** `packages/modules/caching/src/utils/parser.ts`
- **Function:** `buildAffectedCacheKeys`
- **Trigger:** Any `*.updated` event where the update changes filter-relevant fields (status, category, sales channel, etc.)
- **Scope:** Not just products — any entity type with cached list queries that use filters

---

## Reproduction Process

### Environment Setup

*To be filled in during Phase II — will note any issues getting the monorepo running with pnpm.*

### Steps to Reproduce

1. Enable caching in `medusa-config`:
   ```ts
   featureFlags: {
     caching: true,
   }
   ```
2. Create a product in **draft** status
3. Call `GET /store/products` — gets cached, draft product is filtered out
4. Publish the product via the admin API
5. Call `GET /store/products` again

**Result:** Still getting the cached response. Published product doesn't show up.

### Reproduction Evidence

- **Commit showing reproduction:** *Phase II*
- **Screenshots/logs:** *Phase II*
- **My findings:** *To be added after I reproduce it locally*

---

## Solution Approach

### Analysis

The problem is in `buildAffectedCacheKeys` in `parser.ts`. It decides which cache keys to invalidate when an event fires. Right now, it only adds the list-level wildcard (`Entity:list:*`) for `created` and `deleted`:

```ts
if (entity.isInArray || ["created", "deleted"].includes(operation)) {
  keys.add(`${entity.type}:list:*`)
}
```

When `product.updated` fires, it only generates a single-entity key like `Product:prod_xyz`. But since that product was never in the cached list (it was filtered out while in draft), nothing with that tag exists in the cache. The list never gets cleared.

### Proposed Solution

Add `"updated"` to that condition:

```ts
if (entity.isInArray || ["created", "deleted", "updated"].includes(operation)) {
  keys.add(`${entity.type}:list:*`)
}
```

Any update that could affect filter visibility now busts the list cache. Single-entity caches like `Product:prod_123` are untouched.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** On `updated` events, `buildAffectedCacheKeys` skips the list wildcard tag. So cached list responses that filtered out the entity never get invalidated, even if the update changes whether the entity should now be included.

**Match:** The fix follows the same pattern already used for `created` and `deleted` — just extending an array check. I'll look at the existing test file for this function to see how tests are structured before writing new ones.

**Plan:**
1. Open `packages/modules/caching/src/utils/parser.ts`
2. Add `"updated"` to the operation array in `buildAffectedCacheKeys`
3. Find the existing tests (probably `parser.test.ts`)
4. Add test cases for the `updated` scenario
5. Run the test suite locally
6. Manually verify the fix with the reproduction steps above

**Implement:** *Commit links added in Phase III*

**Review:**
- [ ] Follows the project's TypeScript style
- [ ] Existing tests still pass
- [ ] New tests cover the updated behavior
- [ ] PR description clearly explains the problem and fix
- [ ] Checked Medusa's contribution guidelines

**Evaluate:** Run through the reproduction steps again after the fix — draft product published, verify it shows up immediately in `GET /store/products`.

---

## Testing Strategy

### Unit Tests

- [ ] `buildAffectedCacheKeys` with `updated` operation generates `Entity:list:*` tag
- [ ] `buildAffectedCacheKeys` with `updated` still generates the individual entity key (`Entity:id`)
- [ ] `created` and `deleted` operations still behave the same (regression check)

### Integration Tests

- [ ] Caching on: publish a draft product, confirm list cache clears and product appears
- [ ] Caching on: change a product's category, confirm relevant list cache clears

### Manual Testing

*To be filled in during Phase II/III.*

---

## Implementation Notes

### Week 1 Progress

Picked issue #14903 — stale list cache when caching is enabled. The reporter already identified the exact file and function causing it, which made this a solid pick for a first contribution. Commented on the issue to let maintainers know I'm working on it and set up this journal.

### Week 2 Progress

*Coming in Phase II — environment setup and reproduction.*

### Code Changes

- **Files modified:** *Phase III*
- **Key commits:** *Phase III*
- **Approach decisions:** *Will document as I make them*

---

## Pull Request

**PR Link:** *Phase IV*

**PR Description:** *Will draft before submitting*

**Maintainer Feedback:**
- *To be updated*

**Status:** Not yet submitted

---

## Learnings & Reflections

### Technical Skills Gained

*To be filled in at the end.*

### Challenges Overcome

*To be filled in at the end.*

### What I'd Do Differently Next Time

*To be filled in at the end.*

---

## Resources Used

- [medusajs/medusa Issue #14903](https://github.com/medusajs/medusa/issues/14903)
- [Medusa Contribution Guidelines](https://github.com/medusajs/medusa/blob/develop/CONTRIBUTING.md)
- [Medusa Caching Module source](https://github.com/medusajs/medusa/tree/develop/packages/modules/caching)
- *More to be added as I go*
