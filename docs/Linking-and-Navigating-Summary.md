# Linking and Navigating: Client-Side Transitions Summary

> [!NOTE]
> This is a supplemental reading summary for
> [Next.js Docs: Linking and Navigating](https://nextjs.org/docs/app/getting-started/linking-and-navigating#client-side-transitions),
> accompanying [Chapter 5: Navigating Between Pages](https://nextjs.org/learn/dashboard-app/navigating-between-pages)
> of the Next.js Dashboard App tutorial.

## How Navigation Works

Next.js optimizes navigation through four key concepts: **Server Rendering**, **Prefetching**,
**Streaming**, and **Client-side Transitions**.

### Server Rendering

Layouts and Pages are React Server Components by default. The Server Component Payload is generated
on the server before being sent to the client.

- **Prerendering**: Happens at build time or during revalidation; results are cached.
- **Dynamic Rendering**: Happens at request time in response to a client request.

The trade-off: the client must wait for the server to respond before showing a new route. Next.js
addresses this with prefetching and client-side transitions.

### Prefetching

Prefetching loads a route in the background before the user navigates to it. Next.js automatically
prefetches `<Link>` components when they enter the viewport.

- **Static Routes**: The full route is prefetched.
- **Dynamic Routes**: Prefetching is skipped, or the route is **partially prefetched** if
  `loading.tsx` is present.

Partial prefetching avoids unnecessary server work for routes users may never visit.

### Streaming

Streaming allows the server to send parts of a dynamic route to the client as soon as they are
ready, rather than waiting for the entire route to render.

To enable streaming, create a `loading.tsx` in your route folder. Next.js automatically wraps
`page.tsx` contents in a `<Suspense>` boundary. The prefetched fallback UI is shown while the route
loads, then swapped for actual content once ready.

Benefits of `loading.tsx`:

- Immediate navigation and visual feedback.
- Shared layouts remain interactive; navigation is interruptible.
- Improved Core Web Vitals (TTFB, FCP, TTI).

### Client-Side Transitions

Traditionally, navigating to a server-rendered page triggers a full page load, clearing state and
resetting scroll position. Next.js avoids this with the `<Link>` component by:

- Keeping shared layouts and UI intact.
- Replacing the current page with the prefetched loading state or new page.

Next.js also scrolls to the top of the page during transitions. Use CSS `scroll-padding-top` if
content scrolls behind a sticky or fixed header.

## What Can Make Transitions Slow?

### Dynamic Routes Without `loading.tsx`

Without `loading.tsx`, the client waits for the full server response before showing anything,
making the app feel unresponsive. Add `loading.tsx` to enable partial prefetching and show
immediate loading UI.

### Dynamic Segments Without `generateStaticParams`

If a dynamic segment could be prerendered but lacks `generateStaticParams`, it falls back to
dynamic rendering at request time. Add `generateStaticParams` to statically generate routes at
build time.

### Slow Networks

On slow networks, prefetching may not finish before the user clicks a link. Use the
`useLinkStatus` hook to show immediate visual feedback during transitions. You can debounce the
indicator with a short animation delay so it only appears for slower navigations.

### Disabling Prefetching

Set `prefetch={false}` on `<Link>` to opt out. Trade-offs:

- Static routes are only fetched on click.
- Dynamic routes must be rendered on the server before navigation.

To reduce resource usage without fully disabling prefetch, prefetch only on hover by conditionally
setting the `prefetch` prop.

### Hydration Not Completed

`<Link>` is a Client Component and must be hydrated before prefetching starts. Large JavaScript
bundles can delay hydration. Mitigate this by:

- Using `@next/bundle-analyzer` to identify and reduce bundle size.
- Moving logic from client to server where possible.

## Native History API

Next.js integrates with `window.history.pushState` and `window.history.replaceState`, allowing you
to update the browser history without reloading. These calls sync with `usePathname` and
`useSearchParams`.

- **`pushState`**: Adds a new history entry (user can navigate back). Useful for sorting or
  filtering.
- **`replaceState`**: Replaces the current history entry (user cannot navigate back). Useful for
  switching application locale.
