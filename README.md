# nextjs-smooth-navigation

An agent skill that makes Next.js App Router navigation feel instant and smooth.

## What it does

Guides you through implementing a set of navigation optimizations for any Next.js 13+ App Router project:

- **`loading.tsx` skeletons** — page shell renders instantly on click, before any data fetches
- **`router.prefetch()` on hover** — routes pre-warm before the user even clicks
- **Optimistic navbar active state** — active link highlights synchronously on click, no lag
- **`next-view-transitions`** — animated crossfade between pages using the browser's native View Transitions API
- **CSS animation tuning** — control crossfade speed and style

The skill always starts by planning with you — evaluating how many pages you have, whether you have a shared Link wrapper, and which optimizations make sense for your project — before writing any code.

## Install

```bash
npx skills add WingsDevelopment/nextjs-smooth-navigation
```

## Usage

Once installed, ask your AI agent:

> "My Next.js pages feel slow when navigating, can you help?"

> "There's a flash between pages in my Next.js app"

> "Add smooth page transitions to my app"

The skill will walk you through a planning checklist and implement only what you need.

## Requirements

- Next.js 13+ with App Router
- React 18+

## License

MIT
