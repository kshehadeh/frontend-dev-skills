# Core Concepts

## What `cacheComponents: true` Changes

With `cacheComponents` enabled in `next.config.ts`, Next.js 16 fundamentally changes how prerendering works:

**Before (default):** Routes are either statically prerendered or dynamically rendered based on route segment config (`dynamic`, `revalidate`) and whether dynamic APIs are detected.

**After (`cacheComponents: true`):** ALL routes are treated as having dynamic potential. Every async operation in server components must be **explicitly** handled or the build fails with:

```
Error: Route "/[locale]/...": Uncached data or connection() was accessed outside of <Suspense>
```

## The Three Handling Strategies

Every async operation in a server component must use one of:

### 1. `"use cache"` Directive

Caches the function output across requests. The cached result becomes part of the static prerender shell.

```tsx
async function getCachedData(locale: string) {
	'use cache'
	cacheLife('content')
	cacheTag('my-data', `my-data-${locale}`)
	return await fetchData(locale)
}
```

**Use when:** Data is shared across users, changes infrequently, and is SEO-relevant.

**Cannot contain:** `cookies()`, `headers()`, `draftMode()`, `searchParams`, or `connection()`.

### 2. `<Suspense>` Boundary

Defers the async work to request time. The fallback is included in the static shell; the actual content streams in.

```tsx
import { Suspense } from 'react'

export default function Page() {
	return (
		<Suspense fallback={<Loading />}>
			<AsyncContent />
		</Suspense>
	)
}
```

**Use when:** Data is user-specific, request-specific, or not SEO-relevant.

### 3. Synchronous Operations

Purely synchronous server components are automatically prerendered without any special handling.

```tsx
export default function Page() {
	return <div>Static content</div>
}
```

## Cache Configuration

Cache behavior is controlled via `cacheLife()` and `cacheTag()` inside `"use cache"` functions.

### Cache Profiles

Defined in `next.config.ts` under `cacheLife`:

| Profile   | Stale  | Revalidate | Expire | Use Case                    |
| --------- | ------ | ---------- | ------ | --------------------------- |
| `content` | 5 min  | 1 hour     | 1 day  | CMS content, landing pages  |
| `product` | 1 min  | 1 hour     | 1 day  | Product detail pages        |
| `social`  | 1 hour | 1 day      | 1 week | Ratings, reviews            |
| `static`  | 1 hour | 1 day      | 1 week | Size guides, static content |
| `token`   | 1 min  | 1 hour     | 5 days | Auth tokens                 |

### Usage

```tsx
import { cacheLife, cacheTag } from 'next/cache'

async function getProduct(id: string) {
	'use cache'
	cacheLife('product')
	cacheTag('product', `product-${id}`)
	return await fetchProduct(id)
}
```

### Revalidation

Use `revalidateTag()` in server actions or route handlers:

```tsx
'use server'
import { revalidateTag } from 'next/cache'

export async function updateProduct(id: string) {
	await saveProduct(id)
	revalidateTag(`product-${id}`)
}
```

## Key Rules

1. **`"use cache"` caches function OUTPUT** — internal fetches don't need to be independently cacheable
2. **Dynamic values must be resolved BEFORE entering a cache boundary** — pass them as function arguments
3. **`"use cache"` and `<Suspense>` are mutually exclusive strategies** — a single async operation uses one or the other, not both
4. **`generateStaticParams` provides build-time sample values** — these are actually rendered at build time, so they must be valid
5. **Route segment configs (`dynamic`, `revalidate`, `fetchCache`) are obsolete** — remove them when migrating to cacheComponents
