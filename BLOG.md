# I Built a Skill That Makes Next.js Navigation Feel Instant

Written by Srdjan Rakic (@CowboyDev)

---

I built this: `npx skills add WingsDevelopment/nextjs-smooth-navigation`

You run it. Your AI agent asks you a few questions about your app. Then it fixes your navigation — in minutes, not half a day.

Here's what it solves.

---

## The Problem

Click a link in a typical Next.js app. You get:

- A blank screen while data loads
- The navbar highlighting the wrong item for a frame or two
- No transition — just a hard cut between pages

None of this is a bug. It's just what Next.js does by default. And most teams never fix it because — honestly — it's tedious to do right.

---

## Four Fixes. Done Once.

**`loading.tsx`** — Next.js renders this file *synchronously* on click, before any data fetches. The user sees the full page shell instantly. Data fills in underneath.

The catch: it has to match your real page exactly. Same layout. Same header height. Same background. Get it wrong and you trade a blank flash for a layout shift — which is worse.

**`router.prefetch()` on hover** — By the time the user clicks, the route is already loaded.

```tsx
onMouseEnter={() => router.prefetch(href)}
```

**Optimistic navbar active state** — `usePathname()` updates 1–2 frames after navigation starts. Fix: set `isPending` on click, clear it when the route confirms.

```tsx
const active = pathname === url || isPending;
onClick={() => setIsPending(true)}
```

**`next-view-transitions`** — Animated crossfade using the browser's native View Transitions API. Wrap your layout, swap one import, done.

---

## Why a Skill?

Because nobody remembers how to do this.

You've solved it before — on some project, two years ago. But the details don't stick. So next time you either dig through old repos hoping to find that one commit, or you ask an AI agent that confidently gives you something half-right and you spend a full day debugging why the layout shifts or the transitions don't fire.

The skill carries the institutional knowledge. It knows the edge cases — the `"use client"` requirement on `loading.tsx`, the layout shift trap, the fact that `ViewTransitions` alone does nothing unless your links use the library's `Link` component too. It asks the right questions about your specific app before touching anything.

No research. No trial and error. Just results.

```bash
npx skills add WingsDevelopment/nextjs-smooth-navigation
```

Works with Claude Code, Cursor, Windsurf — any agent that supports the SKILL.md standard.
