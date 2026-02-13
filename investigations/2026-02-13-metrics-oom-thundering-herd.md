# Investigation: Metrics Pipeline OOM / Thundering Herd on prod-dash

**Date:** 2026-02-13
**Status:** Investigation complete, actionable findings

## Summary

Render instances on `readme-prod-dash` are OOM crashing due to a combination of:
1. Bulk cache expiration in the Metrics SDK causing a thundering herd of api-registry calls
2. Large dereferenced API specs (1.7MB+ responses, up to 300MB+ for specs like DocuSign) being buffered entirely in memory
3. No size limits, jitter, or deduplication on cache refresh requests

## Root Cause Analysis

### The Thundering Herd

The Metrics SDK (Node.js `readmeio` package) caches the project `baseUrl` via `flat-cache` with a hard-coded **86,400-second (24-hour) TTL**. Cache refresh is **lazy** — triggered only by incoming API requests, with no background scheduler.

When a deploy clears the file-based cache (`node_modules/.cache/readmeio/`), all projects get fresh cache entries at the same timestamp. ~24 hours later, they all expire simultaneously. The next wave of API traffic triggers concurrent fetches to `dash.readme.com/api/v1/` for every project — a classic thundering herd.

**There is no jitter, stagger, mutex, or deduplication** on these fetches.

### The Memory Problem

The SDK monkey-patches `res.write()` and `res.end()` to capture entire response bodies in memory (`patch-response.ts`). The source code itself acknowledges: *"this may take up lots of memory on big responses"*.

When api-registry serves large dereferenced specs (1.7MB for MCP info, potentially 300MB+ for specs like DocuSign), each response is fully buffered in memory. Multiple concurrent large responses during a thundering herd directly cause OOM crashes.

### The Mystery User Agent

The user agent `readme-metrics/2.81.0` does **not match any published SDK version**. No SDK across any registry (npm, PyPI, RubyGems, NuGet, Packagist) has ever had version 2.81.0. This is likely an internal service or custom fork that needs to be identified.

## Answers to Key Questions

### 1. Does the Metrics pipeline have scheduled/cron-based cache refresh?

**No.** Cache refresh is lazy — only triggered by incoming API requests. No cron, timer, or background job.

### 2. What's the cache TTL?

**86,400 seconds (24 hours).** Hard-coded in `get-project-base-url.ts`. On fetch failure, a 2-minute retry timer is set.

### 3. Could bulk cache expiration cause a thundering herd?

**Yes.** A deploy that clears cache → all entries created at same time → all expire ~24hrs later → simultaneous re-fetches. No protection exists.

### 4. Can we add jitter or stagger?

**Not currently built in.** Options ranked by effort:

| Approach | Effort | Impact |
|---|---|---|
| Set `baseLogUrl` config to skip fetch entirely | None (config change) | Eliminates the problem |
| Add random jitter to TTL (±1hr) | Low | Spreads expiry window |
| Stagger initial cache population | Low | Prevents post-deploy bulk expiry |
| Background refresh before expiry | Medium | Eliminates latency spikes |
| Mutex/lock on concurrent fetches | Medium | Prevents duplicate requests |

### 5. Can we cap response size or skip dereferencing?

This requires server-side changes on api-registry/prod-dash. Options:

- Cap response size or paginate api-registry responses
- Skip `$ref` dereferencing for specs above a threshold (e.g., >5MB raw)
- Lazy-dereference: only resolve `$ref`s on demand
- Stream large specs instead of buffering full JSON

## Recommended Immediate Actions

1. **Set `baseLogUrl` in SDK config** for services hitting prod-dash — eliminates the cache-fetch cycle entirely with zero code changes
2. **Identify the `readme-metrics/2.81.0` service** — find what internal component uses this user agent
3. **Add response size limits** on api-registry to prevent 300MB+ dereferenced specs from being served
4. **Add jitter to cache TTL** in the SDK (`get-project-base-url.ts`) to prevent future thundering herds

## Longer-Term Fixes

- Implement background cache refresh (refresh before expiry, not after)
- Add a mutex/lock so only one inflight fetch per cache key is allowed
- Add a configurable max response body size to the SDK's response buffering
- Address the deeper dereferencing problem (Jon's work on lazy/bounded dereferencing)

## References

- [readmeio/metrics-sdks](https://github.com/readmeio/metrics-sdks) — SDK source code
- `packages/node/src/lib/get-project-base-url.ts` — cache TTL logic
- `packages/node/src/lib/patch-response.ts` — response body buffering (memory concern)
- Issue [#876](https://github.com/readmeio/metrics-sdks/issues/876) — AbortError from large payloads
- Issue [#159](https://github.com/readmeio/metrics-sdks/issues/159) — Cache modifies node_modules at runtime
- Dec 1 incident — same pattern (same IP, same user agent, same api-registry calls)
