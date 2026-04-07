---
name: nextjs-smooth-navigation
description: Optimizes Next.js App Router navigation to feel instant and smooth. Use when the user says navigation feels slow, pages flash blank or black between routes, active navbar state lags, or they want animated page transitions. Covers loading.tsx skeletons, router.prefetch() on hover, optimistic navbar active state, and next-view-transitions crossfades.
version: 0.1.0
metadata:
  author: euler-xyz
  category: nextjs
  tags:
    - nextjs
    - navigation
    - performance
    - page-transitions
    - loading-skeleton
    - prefetch
    - view-transitions
    - app-router
---

# Next.js Smooth Navigation

Make Next.js App Router navigation feel instant: zero blank-page flashes, instant skeleton shells, pre-warmed routes, optimistic navbar highlights, and optional animated crossfades.

## When to Use

- User says "navigation feels slow" or "there's a flash between pages"
- Pages show blank/black/white for a moment when navigating
- The active navbar item lags behind after clicking a link
- User wants animated page transitions (crossfades, slides)
- Any Next.js 13+ App Router project where navigation UX needs polish

---

## Step 1 — Plan with the user

Before writing any code, ask these questions and record the answers:

**1. How many pages does the app have?**
List the routes (e.g. `/`, `/dashboard`, `/settings`, `/profile`). Each page with async data needs a `loading.tsx`.

**1a. Does each page need SEO?**
For each page, ask: does it need to be indexed by search engines?

- **SEO required** (landing pages, public content) → the page should be a Server Component that fetches data on the server. `loading.tsx` works perfectly alongside it — crawlers hit `page.tsx` directly and never see `loading.tsx`, so SEO is unaffected. `loading.tsx` is only shown during client-side navigation.
- **SEO not required** (auth-gated dashboards, user-specific data) → the page can be purely client-side (fetching in `useQuery` / hooks). A `loading.tsx` still helps with the blank navigation flash, but the component manages its own loading state internally. In this case a minimal `loading.tsx` (just the page shell/layout, no skeleton rows) is sufficient.
- **No `loading.tsx` at all** — valid choice if the user explicitly wants to skip it for a page. The blank flash remains but nothing breaks.

> Key rule: never add `loading.tsx` to a page that previously rendered on the server if doing so requires converting it to client-only. That would break SEO. Keep Server Components server-side; `loading.tsx` sits beside them, not instead of them.

**2. How complex is each page's loading state?**
- Simple: just a spinner or blank shell → minimal `loading.tsx`
- Medium: page has a header + table → render header + skeleton rows
- Complex: page has tabs, charts, sidebars → decide which parts to skeleton

**3. Does the app have a shared `Link` wrapper component?**
- YES → prefetch and view transitions can be added in ONE place
- NO → must update every nav link individually; consider creating a wrapper first

**4. Where are the nav links? (navbar, sidebar, breadcrumbs, inline links?)**
Count how many separate locations contain navigation links. This determines scope.

**5. Which optimizations does the user want?**
Present as a checklist and let them pick:
- [ ] A) `loading.tsx` — instant page skeleton on click
- [ ] B) `router.prefetch()` on hover — pre-warm routes before click
- [ ] C) Optimistic navbar active state — highlight updates before React confirms route
- [ ] D) `next-view-transitions` — animated crossfade between pages
- [ ] E) CSS transition duration tuning — control crossfade speed

Present a summary plan and confirm before implementing anything.

---

## Step 2 — loading.tsx (Option A)

### Why it works
Next.js App Router uses `loading.tsx` as an instant Suspense boundary. It renders **synchronously** on click, before any server data fetches. The user sees the page shell immediately instead of a blank screen.

### Rules
- Must be a **Client Component** (`"use client"`) if it passes any function props to child components
- Must match the **exact visual shape** of the loaded page — same header height, same layout — to avoid layout shift when real data arrives
- Use the same background class as the real page (e.g. `[background:var(--page-portfolio-bg)]`)
- Pass `isLoading: true` / `isPending: true` via `queryMeta` so the page component renders skeleton rows

### Pattern
```tsx
"use client";

import { SomePage } from "./components/SomePage";

export default function SomeRouteLoading() {
  return (
    <SomePage
      headerData={{
        title: "Page Title",          // exact same title as real page
        description: "Description",   // exact same description — prevents height shift
        metrics: [],                  // empty = no metric values shown
      }}
      filterOptions={{ assets: [], markets: [] }}
      rows={[]}
      queryMeta={{ isLoading: true, isPending: true }}
    />
  );
}
```

### What to check in the real PageClient
- Find the `FALLBACK_HEADER` / `FALLBACK_*` constants — use those exact values
- Check the exact prop names: some pages use `queryMeta`, others split into `headerQueryMeta` + `tableQueryMeta`
- Check `filterOptions` key names — they differ per page (e.g. `assets` vs `collateralAssets`)
- Exclude `loading.tsx` and `*.stories.tsx` from TypeScript compilation in `tsconfig.json`:
  ```json
  "exclude": ["node_modules", "**/*.stories.ts", "**/*.stories.tsx"]
  ```

### One loading.tsx per dynamic page
Create `loading.tsx` in each page directory that fetches data:
```
app/
  dashboard/
    loading.tsx   ← new
    page.tsx
  settings/
    loading.tsx   ← new
    page.tsx
```
Static pages (no async data) don't need one.

---

## Step 3 — router.prefetch() on hover (Option B)

### Why it works
`router.prefetch()` pre-warms the route segment (including the `loading.tsx` boundary) before the user clicks. By the time they click, the shell renders in 0ms.

### If there is a shared Link wrapper — one change covers all nav links
```tsx
"use client";
import { useRouter } from "next/navigation";
import Link from "next/link";

export function NavLink({ href, children, ...props }) {
  const router = useRouter();
  return (
    <Link
      href={href}
      onMouseEnter={() => router.prefetch(href)}
      {...props}
    >
      {children}
    </Link>
  );
}
```

### If nav links are scattered — update each location
Add `onMouseEnter={() => router.prefetch(url)}` to every `<Link>` or `<a>` that navigates between app pages. Skip external links.

### Mobile (touch devices)
Touch devices don't have hover. Prefetch on `onFocus` as well, or add a short `onTouchStart` handler:
```tsx
onTouchStart={() => router.prefetch(href)}
```

---

## Step 4 — Optimistic navbar active state (Option C)

### Why it works
`usePathname()` is a `useSyncExternalStore` read inside React's concurrent transition. It updates 1–2 frames after navigation, causing a visible lag where the navbar still shows the old active item. Fix: set local `isPending` state **synchronously on click**, clear it when `usePathname()` confirms the new route.

### Pattern for a nav item component
```tsx
"use client";
import { usePathname, useRouter } from "next/navigation";
import * as React from "react";

function NavItem({ url, children }) {
  const pathname = usePathname();
  const router = useRouter();
  const [isPending, setIsPending] = React.useState(false);

  // Clear pending when route actually changes
  React.useEffect(() => {
    setIsPending(false);
  }, [pathname]);

  const active = pathname === url || isPending;

  return (
    <Link
      href={url}
      onMouseEnter={() => router.prefetch(url)}
      onClick={() => setIsPending(true)}
      data-active={active}
    >
      {children}
    </Link>
  );
}
```

### If the navbar uses a flat list of link objects (GlobalNavbar pattern)
```tsx
const [pendingHref, setPendingHref] = React.useState<string | null>(null);

React.useEffect(() => {
  setPendingHref(null);
}, [pathname]);

// per link:
const active = isLinkActive(pathname, link.href) || pendingHref === link.href;

// on the link element:
onMouseEnter={() => router.prefetch(link.href)}
onClick={() => setPendingHref(link.href)}
```

---

## Step 5 — next-view-transitions (Option D + E)

### Install
```bash
pnpm add next-view-transitions
# or: npm install next-view-transitions
```

### Step 1: Wrap root layout
```tsx
// app/layout.tsx
import { ViewTransitions } from "next-view-transitions";

export default function RootLayout({ children }) {
  return (
    <ViewTransitions>
      <html lang="en">
        <body>{children}</body>
      </html>
    </ViewTransitions>
  );
}
```

### Step 2: Use the library's Link (triggers the transition)
The `ViewTransitions` provider alone does nothing — navigation must use the library's `Link` (or `useTransitionRouter`) to fire the transition.

**If there is a shared Link wrapper** — one import change:
```tsx
// Before:
import Link from "next/link";
// After:
import { Link } from "next-view-transitions";
```

**If using `useRouter` for programmatic navigation:**
```tsx
import { useTransitionRouter } from "next-view-transitions";

const router = useTransitionRouter(); // replaces useRouter()
router.push("/dashboard");            // triggers crossfade
```

### Step 3: Tune the animation (Option E)
Default crossfade is ~250ms. Customize in `globals.css`:
```css
/* Faster crossfade (snappier feel) */
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 150ms;
}

/* Slide instead of crossfade */
::view-transition-old(root) {
  animation: 150ms ease-in both slide-out-to-left;
}
::view-transition-new(root) {
  animation: 150ms ease-out both slide-in-from-right;
}
```

### Browser support
Supported in Chrome 111+, Edge 111+, Safari 18.2+. Falls back gracefully (instant navigation, no transition) in Firefox and older browsers.

---

## Quick reference — what to do for a new project

| Situation | Action |
|-----------|--------|
| Pages flash blank on navigation | Add `loading.tsx` to each data-fetching page |
| `loading.tsx` causes layout shift | Match description text exactly, match background class |
| `loading.tsx` crashes with "Event handlers cannot be passed to Client Component" | Add `"use client"` to `loading.tsx` |
| Navbar active state lags | Add `isPending` state + `useEffect` to clear on pathname change |
| Routes feel slow even with loading.tsx | Add `router.prefetch()` on `onMouseEnter` |
| User wants animated transitions | Install `next-view-transitions`, wrap layout, swap `Link` import |
| Transitions not working | Check that nav links use `next-view-transitions`'s `Link`, not `next/link` |
| Black/white flash on specific page | That page has a `<Suspense fallback={null}>` — remove it and let `loading.tsx` handle it |
| TypeScript errors in `loading.tsx` | Inline the fallback values — don't export them from PageClient |
| Page needs SEO — should I add `loading.tsx`? | Yes — `loading.tsx` only shows during client navigation, never to crawlers. Keep `page.tsx` as a Server Component; `loading.tsx` sits beside it safely |
| Page is client-only (no SEO needed) — should I add `loading.tsx`? | Optional. A minimal shell `loading.tsx` still removes the blank flash. The component handles its own skeleton rows internally |
| User wants to skip `loading.tsx` on a specific page | Fine — skip it. The navigation flash stays but nothing breaks. Not every page needs it |
