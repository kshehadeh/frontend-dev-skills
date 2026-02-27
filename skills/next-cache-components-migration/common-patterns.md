# Common Patterns & Fixes

## Pattern: Async Server Component Provider Wrapper

### Problem

An async server component fetches data and wraps `{children}` in context providers. With `cacheComponents`, this blocks the entire route because the layout's async work is neither cached nor in Suspense.

```tsx
// DynamicProviders.tsx — PROBLEMATIC
export default async function DynamicProviders({ children }: { children: React.ReactNode }) {
	const config = await fetchConfig() // Uncached async
	const flags = await initializeFlags() // Accesses cookies/draftMode
	return (
		<ConfigProvider config={config}>
			<FlagProvider flags={flags}>{children}</FlagProvider>
		</ConfigProvider>
	)
}
```

### Fix

Split into three parts:

1. **Cached data function** (`layout-data.ts`) — uses `"use cache"`
2. **Synchronous provider wrapper** (`Providers.tsx`) — receives data as props
3. **Layout** — calls cached function, passes result to sync wrapper

```tsx
// layout-data.ts
import { cacheLife, cacheTag } from 'next/cache'

export async function getCachedLayoutData(locale: string) {
	'use cache'
	cacheLife('content')
	cacheTag('layout-data', `layout-${locale}`)
	const config = await fetchConfig(locale)
	return { config }
}

// Providers.tsx — SYNCHRONOUS, no async work
;('use client')
export function Providers({ config, children }: { config: Config; children: React.ReactNode }) {
	return <ConfigProvider config={config}>{children}</ConfigProvider>
}

// layout.tsx
import { getCachedLayoutData } from './layout-data'
import { Providers } from './Providers'

export default async function Layout({ children, params }) {
	const { locale } = await params
	const { config } = await getCachedLayoutData(locale)
	return <Providers config={config}>{children}</Providers>
}
```

---

## Pattern: Server Actions Passed to Client Components

### Problem

Importing a `'use server'` function and passing it as a prop to a client component.

### Verdict

This is **fine**. Server action references are serializable. If the error trace points to the client component receiving the action, the real issue is usually elsewhere in the render tree — look at the server component boundary above it.

---

## Pattern: Module-Level Dynamic API Imports

### Problem

A file imports `cookies`/`draftMode`/`headers` at the top level, even though a specific function doesn't call them.

```tsx
import { cookies, draftMode } from 'next/headers'

// This function is safe
export async function getPublicData() {
	'use cache'
	return await fetchPublicContent()
}

// This function uses cookies — NOT safe for cache
export async function getUserData() {
	const token = (await cookies()).get('token')
	return await fetchUserContent(token)
}
```

### Fix

The import itself doesn't cause issues. The problem is when `getUserData()` is called outside a cache or Suspense boundary. Create separate cache-safe functions that don't call dynamic APIs:

```tsx
// For cache-safe flag checking, create a dedicated helper
export async function hasFlagCached(flagName: string): Promise<boolean> {
	'use cache'
	cacheLife('content')
	const flags = await fetchFlagsFromCMS() // no cookies/draftMode
	return flags.includes(flagName)
}
```

---

## Pattern: Error Handling with Logging Before Data Access

### Problem

`logger.error(err, 'message')` calls `Date.now()` internally (e.g., pino timestamps). If this happens before any cached data access in the component tree, it triggers the non-deterministic operation error.

```tsx
// generateStaticParams returns placeholder values
// → page renders with placeholder → data not found → catch block → logger.error() → Date.now() → BUILD FAILS
export default async function Page(props) {
	const { id } = await props.params
	try {
		const data = await getCachedData(id) // "use cache" inside
		return <Content data={data} />
	} catch (err) {
		logger.error(err, 'Page failed') // Date.now() before any successful cached access
		return notFound()
	}
}
```

### Fix Options

1. **Move error handling inside the cached function** — `Date.now()` inside `"use cache"` is cached and safe
2. **Use `console.error()` instead** in the outer catch (no timestamp injection)
3. **Ensure `generateStaticParams` values are valid** — so the error path is never hit during build
4. **Wrap in `<Suspense>`** — non-deterministic operations inside Suspense are deferred to request time

---

## Pattern: `React.cache()` + `"use cache"` Together

### Why Both?

They serve different purposes and can coexist:

- **`React.cache()`** — per-request deduplication. Multiple components calling the same function in one render get the same result from memory.
- **`"use cache"`** — cross-request caching. Results cached to disk/CDN, served across different user requests.

```tsx
import { cache } from 'react'
import { cacheLife, cacheTag } from 'next/cache'

async function fetchProductImpl(id: string) {
	'use cache'
	cacheLife('product')
	cacheTag(`product-${id}`)
	return await api.getProduct(id)
}

// Wrapping with React.cache() deduplicates within a single request render
export const getProduct = cache(fetchProductImpl)
```

---

## Pattern: Parallel Route `default.tsx` Files

### Problem

Parallel route `default.tsx` files (e.g., `@header/default.tsx`, `@sidebar/default.tsx`) are **always rendered by Next.js** during prerendering, even if the layout doesn't explicitly reference them in JSX.

### Fix

Ensure all parallel route `default.tsx` files either:

1. Use `"use cache"` for their data fetching:

   ```tsx
   // @header/default.tsx
   async function getCachedHeader(locale: string) {
   	'use cache'
   	cacheLife('content')
   	return await fetchHeaderData(locale)
   }

   export default async function HeaderDefault(props) {
   	const { locale } = await props.params
   	const data = await getCachedHeader(locale)
   	return <Header data={data} />
   }
   ```

2. Or wrap content in `<Suspense>`:
   ```tsx
   export default function HeaderDefault() {
   	return (
   		<Suspense fallback={<HeaderSkeleton />}>
   			<HeaderContent />
   		</Suspense>
   	)
   }
   ```

---

## Pattern: Feature Flag Checks Inside Cache Boundaries

### Problem

Feature flag APIs that rely on `cookies()` or `draftMode()` cannot be called inside `"use cache"`.

```tsx
// BAD
async function getCachedContent() {
	'use cache'
	const showFeature = await hasFlag('new-feature') // Uses draftMode() internally
	return showFeature ? <NewFeature /> : <OldFeature />
}
```

### Fix

Create a cache-safe flag checker that fetches flags from the CMS without using runtime APIs:

```tsx
export async function hasFlagCached(flagName: string): Promise<boolean> {
	'use cache'
	cacheLife('content')
	cacheTag('flags')
	const flags = await fetchFlagsFromAPI() // Direct API call, no cookies/draftMode
	return flags.includes(flagName)
}
```

Or check the flag **outside** the cache boundary and pass the result in:

```tsx
async function getCachedContent(showFeature: boolean) {
	'use cache'
	return showFeature ? await fetchNewContent() : await fetchOldContent()
}

export default async function Page() {
	const showFeature = await hasFlagCached('new-feature')
	const content = await getCachedContent(showFeature)
	return <Content data={content} />
}
```
