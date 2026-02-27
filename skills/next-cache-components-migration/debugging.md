# Debugging `cacheComponents` Build Errors

## Step 1: Get the Error Details

Run the build to identify failing routes:

```bash
next build
```

For detailed stack traces:

```bash
next build --debug-prerender
```

The error message includes the route path:

```
Error: Route "/[locale]/(layout-client)/(full-navbar)/product/[searchParams]/style/[...productId]":
  Uncached data or connection() was accessed outside of <Suspense>
```

## Step 2: Read the Stack Trace Carefully

**Warning:** Stack traces often show **client components** — these are misleading. The real issue is in the **server component boundary above them** in the render tree.

Look for:

- The server component that renders the client component
- Any `await` calls in server components outside `"use cache"` or `<Suspense>`
- Imports from modules that access `cookies()`, `headers()`, or `draftMode()`

## Step 3: Systematic Elimination

If the error persists after fixing the obvious issue, isolate by commenting out sections:

### Level 1: Is it the page or the layout?

```tsx
// Replace page content with static HTML
export default function Page() {
	return <div>test</div>
}
```

- **Error gone:** Issue is in the page component
- **Error persists:** Issue is in a layout, parallel route, or provider

### Level 2: Is it in layouts or parallel routes?

```tsx
// Strip layout to bare minimum
export default function Layout({ children }) {
	return <div>{children}</div>
}
```

- **Error gone:** Issue is in layout providers/data fetching
- **Error persists:** Check parent layouts or parallel route slots

### Level 3: Which parallel route?

Comment out parallel route slots one at a time in the layout:

```tsx
export default function Layout({
	children,
	// header,      // Comment out one at a time
	// sidebar,
}) {
	return <div>{children}</div>
}
```

Note: Even commented out in JSX, `default.tsx` files in parallel routes **still render** during prerendering. You may need to make the `default.tsx` return static content to isolate.

### Level 4: Which provider?

Add providers back one at a time:

```tsx
export default async function Layout({ children }) {
	// const config = await getCachedLayoutData(locale)
	return (
		// <ConfigProvider config={config}>
		//   <FlagProvider>
		{ children }
		//   </FlagProvider>
		// </ConfigProvider>
	)
}
```

## Step 4: Check for Hidden Non-Deterministic Operations

Common sources of hidden `Date.now()` or `new Date()`:

| Source                               | Why                                        | Fix                                                |
| ------------------------------------ | ------------------------------------------ | -------------------------------------------------- |
| `logger.error()` / `logger.warn()`   | Pino/Winston add timestamps                | Use `console.error()` or move inside `"use cache"` |
| `new Error('...')` with stack traces | Some environments add timestamps           | Usually fine, check custom Error subclasses        |
| Analytics/tracking calls             | Often include timestamps                   | Move after cached data access                      |
| UUID generation                      | `crypto.randomUUID()` is non-deterministic | Move after cached data access or inside cache      |

## Step 5: Validate `generateStaticParams` Values

If using `generateStaticParams`, verify the sample values don't trigger error paths:

1. Check that the sample value exists in your data source (or that the page handles "not found" gracefully)
2. Check that error handlers in the render path don't use timestamp-dependent logging
3. Test by temporarily hardcoding the sample value and running the build

```tsx
// Test: Does this route build with the sample value?
export async function generateStaticParams() {
	return [{ slug: 'your-sample-value' }]
}
```

If the sample triggers a `notFound()` call that logs with timestamps, the build will fail.

## Common Error Patterns and Their Causes

### "Used Date.now() before accessing uncached data"

A non-deterministic operation happened before any `"use cache"` or `<Suspense>` boundary was entered. Check the render path for:

- Logger calls in error handlers
- Timestamp generation for analytics
- Random ID generation

### "Route ... A]": Uncached data was accessed outside of <Suspense>"

The route has an async server component that does I/O without `"use cache"` or `<Suspense>`. Check:

- Layout files in the route's path
- Parallel route `default.tsx` files
- The page component itself

### Build passes locally but fails in CI

Check for:

- Different environment variables (feature flags, API endpoints)
- Different Node.js versions
- Race conditions in data fetching that cause intermittent errors with sample `generateStaticParams` values
