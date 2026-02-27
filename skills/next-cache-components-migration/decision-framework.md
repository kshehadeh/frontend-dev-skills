# Decision Framework

When encountering the build error `Error: Route "...": Uncached data or connection() was accessed outside of <Suspense>`, determine which category the problematic code falls into and apply the corresponding fix.

## Category 1: SEO-Sensitive Pages with Data Fetching

Pages like product detail, landing pages, category pages where content must be in the initial HTML for search engines.

### Fix

Use `"use cache"` on the data-fetching function or component:

```tsx
import { cacheLife, cacheTag } from 'next/cache'

async function getCachedPageData(slug: string) {
	'use cache'
	cacheLife('content')
	cacheTag('page', `page-${slug}`)
	return await fetchPageContent(slug)
}

export default async function LandingPage(props: { params: Promise<{ slug: string }> }) {
	const { slug } = await props.params
	const data = await getCachedPageData(slug)
	return <PageContent data={data} />
}
```

If the page has dynamic route segments, provide `generateStaticParams` with **valid** sample values:

```tsx
export async function generateStaticParams() {
	return [{ slug: 'homepage' }]
}
```

The sample values **will be rendered at build time** — they must exercise the happy path without errors.

## Category 2: Non-SEO Pages

Auth-gated pages (user dashboard, order history), transactional pages (checkout), utility pages (download-invoice, print labels).

### Fix

Wrap page content in `<Suspense>`:

```tsx
import { Suspense } from 'react'

async function OrderContent() {
	const order = await fetchOrder()
	return <OrderDetails order={order} />
}

export default function OrderPage() {
	return (
		<Suspense fallback={<OrderSkeleton />}>
			<OrderContent />
		</Suspense>
	)
}
```

The page's default export should be a **synchronous** thin wrapper.

## Category 3: Pages Accessing Runtime Data

Pages using `cookies()`, `headers()`, `draftMode()`, or `searchParams`.

### Fix

Runtime APIs **CANNOT** be inside `"use cache"`. Extract runtime data access into a child component wrapped in `<Suspense>`:

```tsx
// BAD — will error
async function getCachedData() {
	'use cache'
	const token = (await cookies()).get('token') // VIOLATION
	return await fetchWithToken(token)
}

// GOOD — extract runtime, pass to cached function
async function getCachedData(locale: string) {
	'use cache'
	cacheLife('content')
	return await fetchContent(locale)
}

async function PageContent(props: { searchParams: Promise<Record<string, string>> }) {
	const params = await props.searchParams
	const locale = params.locale ?? 'en'
	const data = await getCachedData(locale)
	return <Content data={data} query={params.q} />
}

export default function Page(props: { searchParams: Promise<Record<string, string>> }) {
	return (
		<Suspense fallback={<Loading />}>
			<PageContent searchParams={props.searchParams} />
		</Suspense>
	)
}
```

You **CAN** pass resolved runtime values as arguments to cached functions — the cache key will include those argument values.

## Category 4: Dynamic Route Params

Routes with `[slug]`, `[...path]`, or `[[...optional]]` segments.

### Strategy A: With `generateStaticParams` (preferred for SEO pages)

Provide at least one valid sample per dynamic segment. `await params` is allowed directly in the component because Next.js validates against the sample values at build time.

```tsx
export async function generateStaticParams() {
	return [{ slug: 'sample-product' }]
}

export default async function ProductPage(props: { params: Promise<{ slug: string }> }) {
	const { slug } = await props.params
	const product = await getCachedProduct(slug)
	return <Product data={product} />
}
```

**Requirements for sample values:**

- Must not trigger unhandled errors in data fetching
- Must not call `Date.now()` or `new Date()` before accessing cached/dynamic data
- Must not use `logger.error()` (internally uses timestamps) before cached data access
- Must handle gracefully if data doesn't exist (e.g., `notFound()` without logging)

**Segment type examples:**

| Segment Type       | Pattern       | generateStaticParams       |
| ------------------ | ------------- | -------------------------- |
| Single dynamic     | `[slug]`      | `[{ slug: 'sample' }]`     |
| Catch-all          | `[...path]`   | `[{ path: ['segment1'] }]` |
| Optional catch-all | `[[...path]]` | `[{ path: ['segment1'] }]` |

### Strategy B: Without `generateStaticParams` (non-SEO pages)

All param access becomes runtime data. Wrap in `<Suspense>`:

```tsx
async function Content(props: { params: Promise<{ orderId: string }> }) {
	const { orderId } = await props.params
	const order = await fetchOrder(orderId)
	return <OrderDetails order={order} />
}

export default function OrderPage(props: { params: Promise<{ orderId: string }> }) {
	return (
		<Suspense fallback={<OrderSkeleton />}>
			<Content params={props.params} />
		</Suspense>
	)
}
```

## Category 5: Layouts

Layouts that `await params` or fetch data asynchronously.

### Key Constraints

- Layouts **CANNOT** use `"use cache"` because they accept `children` (a non-serializable prop)
- If the layout's segment has `generateStaticParams`, `await params` is OK
- Data fetching must be extracted to a separate `"use cache"` function

### Fix: Split Data Fetching from Rendering

```tsx
// layout-data.ts
import { cacheLife, cacheTag } from 'next/cache'

export async function getCachedLayoutData(locale: string) {
	'use cache'
	cacheLife('content')
	cacheTag('layout', `layout-${locale}`)
	return await fetchLayoutConfig(locale)
}

// layout.tsx
import { getCachedLayoutData } from './layout-data'
import { Providers } from './Providers'

export default async function Layout({
	children,
	params,
}: {
	children: React.ReactNode
	params: Promise<{ locale: string }>
}) {
	const { locale } = await params
	const config = await getCachedLayoutData(locale)
	return <Providers config={config}>{children}</Providers>
}
```

The `Providers` component should be **synchronous** — it receives pre-fetched data as props.

## Category 6: Non-Deterministic Operations

`Date.now()`, `new Date()`, `Math.random()`, `crypto.randomUUID()`.

### Rule

Non-deterministic operations must come **AFTER** dynamic/cached data access, never before.

```tsx
// BAD — Date.now() before cached data access
export default async function Page() {
	const timestamp = Date.now() // VIOLATION
	const data = await getCachedData()
	return <div>{data}</div>
}

// GOOD — after cached data access
export default async function Page() {
	const data = await getCachedData()
	const timestamp = Date.now()
	return <div>{data}</div>
}
```

### Common Gotcha: Logging Libraries

`logger.error()` (e.g., pino) internally calls `Date.now()` for timestamps. If logging happens in error handlers **before** any cached data access, it triggers the error:

```tsx
// BAD — logger.error uses timestamps
async function fetchData() {
	'use cache'
	try {
		return await apiCall()
	} catch (err) {
		logger.error(err, 'Failed') // Uses Date.now() internally — OK inside "use cache"
		return null
	}
}
```

Inside `"use cache"`, `Date.now()` executes once and the result is cached — this is fine. The problem occurs when logging is **outside** any cache or Suspense boundary.

### Fix for Outside Boundaries

Use `await connection()` from `next/server` before non-deterministic operations, and wrap in `<Suspense>`:

```tsx
import { connection } from 'next/server'

async function DynamicContent() {
	await connection()
	const now = Date.now()
	// ...
}

export default function Page() {
	return (
		<Suspense fallback={null}>
			<DynamicContent />
		</Suspense>
	)
}
```
