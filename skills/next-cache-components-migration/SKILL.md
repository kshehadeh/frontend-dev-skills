---
name: next-cache-components-migration
description: Next.js 16 cacheComponents diagnostics - resolving "Uncached data outside Suspense" errors, generateStaticParams requirements, cache boundary violations, migration patterns
user-invocable: false
---

# Next.js Cache Components (`cacheComponents: true`)

Diagnose and fix build errors when `cacheComponents: true` is enabled in `next.config.ts`.

## Core Concepts

See [core-concepts.md](./core-concepts.md) for:

- How cacheComponents changes prerendering behavior
- The three handling strategies (cache, Suspense, synchronous)
- Cache profiles and configuration (`cacheLife`, `cacheTag`)

## Decision Framework

See [decision-framework.md](./decision-framework.md) for:

- Category 1: SEO-sensitive pages with data fetching
- Category 2: Non-SEO pages (auth-gated, transactional, utility)
- Category 3: Pages accessing runtime data (cookies, headers, searchParams)
- Category 4: Dynamic route params (`[slug]`, `[...path]`, `[[...optional]]`)
- Category 5: Layouts
- Category 6: Non-deterministic operations (`Date.now()`, `Math.random()`)

## Common Patterns & Fixes

See [common-patterns.md](./common-patterns.md) for:

- Async server component provider pattern
- Server actions passed to client components
- Module-level dynamic API imports
- Error handling with logging before data access
- `React.cache()` + `"use cache"` coexistence
- Parallel route default.tsx handling

## Debugging

See [debugging.md](./debugging.md) for:

- Using `next build --debug-prerender` for stack traces
- Systematic elimination approach
- Misleading client component traces
- Hidden timestamp operations in logging libraries

## Migration Checklist

See [migration-checklist.md](./migration-checklist.md) for:

- Step-by-step checklist for migrating existing pages
- Obsolete config options to remove
- Testing strategy
