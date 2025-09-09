# Using `revalidate = Infinity` with On-Demand Invalidation in Next.js

## Introduction
In Next.js App Router, the `revalidate` property controls how long cached data remains fresh before Next.js attempts to re-fetch it. Setting `revalidate = Infinity` (or equivalently `false`) means **cache forever until explicitly invalidated**. This model shifts responsibility from **time-driven cache refresh** to **event-driven invalidation**.

This article examines whether this is a good practice, the trade-offs involved, and how enterprises can implement it safely.

---

## What the Official Docs Say
- **Default behavior:** `revalidate: false` is the default, which equals `Infinity`—data and routes are cached indefinitely unless explicitly refreshed. [Docs]
- **Time-based revalidation:** You can set `revalidate: N` to refresh after `N` seconds. [Docs]
- **On-demand invalidation:** Next.js provides `revalidatePath` and `revalidateTag` to purge cached data when business events (like CMS updates) occur. The next request fetches fresh data. [Docs]
- **Dynamic routes:** Routes using dynamic functions or `no-store`/`revalidate: 0` bypass caching entirely. [Docs]

---

## When It’s a Good Practice
Setting `revalidate = Infinity` with on-demand invalidation works well when:
- **Data is mostly static** and only changes due to **controlled events** (CMS publish, admin update).
- You can reliably trigger invalidation through webhooks, queues, or mutation handlers.
- Your primary objective is **performance and scalability**, reducing origin load.

**Pros:**
- **Maximum performance:** Uses the Full Route Cache and Data Cache aggressively.
- **Deterministic freshness:** Updates only when real business events happen.
- **Operational simplicity:** No background refresh or race conditions.

---

## Risks and Trade-offs
- **Staleness risk:** If invalidation fails (e.g., webhook outage), stale data persists indefinitely.
- **Complex tagging:** Every cached fetch must be correctly tagged or linked to a path. Missing tags = partial staleness.
- **Not for dynamic content:** Auth dashboards, carts, or real-time feeds should use `no-store` or `revalidate: 0`.
- **Dynamic features may bypass caching:** Functions like cookies or auth headers force dynamic rendering.

---

## Enterprise-Grade Recommendations
1. **Default to Infinity:** Use `revalidate = Infinity` at the segment level; override with `no-store` or `revalidate: 0` where necessary.
2. **Adopt tagging discipline:** Standardize cache tags (e.g., `post:123`, `category:shoes`) and always pair data mutations with `revalidateTag` calls.
3. **Harden invalidation pipelines:**
   - Use retryable queues or job workers.
   - Log every revalidation call.
   - Monitor cache hit/miss ratios.
4. **Fallback with time-based revalidation:** For third-party data without webhooks, set a conservative interval (e.g., `revalidate: 300`).
5. **Audit exceptions:** Keep a list of routes/data marked as `no-store` or `revalidate: 0`, and ensure no sensitive or dynamic data leaks into cached layers.

---

## Bottom Line
- If your system can **guarantee reliable invalidation**, then `revalidate = Infinity + on-demand invalidation` is **optimal for performance and cost**.
- If your invalidation path is brittle, you risk **indefinite staleness**—in which case time-based revalidation or no-store caching is safer.

---

✅ **Recommendation:** For enterprise-grade systems, use **Infinity + on-demand invalidation** as the default, but pair it with strong observability, retries, and disciplined cache tagging.