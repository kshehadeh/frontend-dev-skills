# Migration Checklist

Use this checklist when migrating existing pages to work with `cacheComponents: true`, or when adding new pages to a project that already uses it.

## Pre-Migration

- [ ] Enable `cacheComponents: true` in `next.config.ts`
- [ ] Define cache profiles in `next.config.ts` under `cacheLife`
- [ ] Ensure `next` version is 16+ with `"use cache"` support

## Per-Route Checklist

### Data Fetching

- [ ] Every `async` function doing I/O in server components has `"use cache"` **or** is in `<Suspense>`
- [ ] `"use cache"` functions include `cacheLife()` and `cacheTag()` calls
- [ ] No runtime APIs (`cookies()`, `headers()`, `draftMode()`) inside `"use cache"` boundaries
- [ ] Runtime values needed by cached functions are passed as **arguments**, not accessed internally

### Dynamic Route Segments

- [ ] Every dynamic route segment (`[slug]`, `[...path]`, `[[...optional]]`) has `generateStaticParams` **or** param access is in `<Suspense>`
- [ ] `generateStaticParams` returns at least one valid sample value per segment
- [ ] Sample values exercise the **happy path** — no unhandled errors, no timestamp-dependent logging
- [ ] Catch-all segments use array values: `{ path: ['segment'] }`

### Layouts

- [ ] Layouts do NOT use `"use cache"` (they accept `children`, a non-serializable prop)
- [ ] Layout data fetching is extracted to separate `"use cache"` functions
- [ ] Provider wrappers are **synchronous** — they receive pre-fetched data as props
- [ ] `await params` in layouts is valid only if the segment has `generateStaticParams`

### Parallel Routes

- [ ] All `default.tsx` files in parallel routes handle caching properly
- [ ] Parallel route data fetching uses `"use cache"` or is in `<Suspense>`
- [ ] Remember: `default.tsx` files render even if not referenced in layout JSX

### Non-Deterministic Operations

- [ ] No `Date.now()` / `new Date()` / `Math.random()` **before** first cached/dynamic data access
- [ ] No `logger.error()` / `logger.warn()` (which use timestamps internally) before data access
- [ ] If unavoidable, use `await connection()` + `<Suspense>` to defer

### Error Handling

- [ ] Error handlers that use logging are **inside** `"use cache"` boundaries or `<Suspense>`
- [ ] `generateStaticParams` sample values don't trigger error paths with timestamp logging
- [ ] `notFound()` calls in data fetching don't log before returning

## Cleanup

### Remove Obsolete Route Segment Configs

These are no longer needed with `cacheComponents`:

- [ ] Remove `export const dynamic = 'force-dynamic'` / `'force-static'`
- [ ] Remove `export const revalidate = N`
- [ ] Remove `export const fetchCache = '...'`
- [ ] Remove `export const runtime = 'edge'` (unless actually needed for Edge runtime)

### Replace Deprecated APIs

- [ ] Replace `unstable_cache()` with `"use cache"` functions
- [ ] Replace `unstable_noStore()` with `await connection()` from `next/server`

## Testing Strategy

1. **Build locally:** `next build` to catch all route errors
2. **Use debug mode:** `next build --debug-prerender` for detailed stack traces
3. **Test incrementally:** Fix one route at a time, rebuild to confirm
4. **Verify caching:** Use `next dev` with cache inspector to confirm cache hits
5. **Check revalidation:** Trigger `revalidateTag()` and verify content updates

## Quick Reference: When to Use What

| Scenario                              | Strategy                                           |
| ------------------------------------- | -------------------------------------------------- |
| SEO page with shared data             | `"use cache"` + `cacheLife` + `cacheTag`           |
| User-specific page                    | `<Suspense>`                                       |
| Runtime API access (cookies, headers) | `<Suspense>`, pass values to cached functions      |
| Dynamic route, SEO-important          | `generateStaticParams` + `"use cache"`             |
| Dynamic route, non-SEO                | `<Suspense>` around param access                   |
| Layout data fetching                  | Extract to `"use cache"` function in separate file |
| Feature flag checks                   | Cache-safe helper or check outside cache boundary  |
| Timestamps / random values            | After cached data access, or inside `<Suspense>`   |
