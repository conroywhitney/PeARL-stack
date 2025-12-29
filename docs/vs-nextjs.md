---
layout: page
title: PeARL vs. Next.js + Supabase
permalink: /vs-nextjs/
nav_order: 4
---

An honest comparison. No FUD, just reality.

---

## The Footgun Catalog

Real issues beginners hit with the JS stack:

| JS/Next.js Problem | What Happens | PeARL Equivalent |
|---|---|---|
| `useEffect` infinite loop | App freezes, no obvious error | No useEffect—LiveView handles reactivity |
| Forgot `"use client"` | Cryptic hydration error | No client/server split—it's all server |
| Stale closure in callback | Bug that only appears sometimes | Immutable data—no closures to go stale |
| State update in unmounted component | Memory leak warning spam | LiveView cleans up automatically |
| N+1 queries in RSC | Slow pages, hard to debug | Ash preloading is explicit |
| Vercel bill spike | $500 surprise invoice | Fly.io predictable pricing |
| Next.js upgrade breaking | Half your code needs rewriting | Phoenix upgrades are boring (good boring) |
| "Works locally, breaks in prod" | Environment differences | BEAM is the same everywhere |
| CORS errors | Confusing, random-seeming | Same-origin by default (WebSocket) |
| JWT expiration at midnight | Auth breaks mysteriously | Session-based, server-managed |
| Prisma migration conflicts | Database drift | Ecto migrations are linear |

---

## "But I Already Know JavaScript"

This is the most common objection. Let's be honest about it.

**Reality: You're learning three things pretending to be one.**

With Next.js + React, you're actually learning:

1. **React Client JS** — hooks, state, effects, the rules of hooks
2. **Next.js RSC** — server components, streaming, caching, the `"use client"` boundary
3. **Node.js API routes** — different runtime, different mental model

The `"use client"` boundary alone is a whole new paradigm to understand.

**With PeARL:**

- React is React (you already know this)
- Elixir is Elixir (one new thing)
- The boundary is explicit: `pushEvent` / `handleEvent`

---

## The Three Runtimes Problem

Here's what Next.js actually runs:

```
┌─────────────────────────────────────────────────┐
│                   Your Next.js App               │
├─────────────────────────────────────────────────┤
│  Browser JavaScript     │  "use client" barrier  │
│  (React, hooks, state)  │                        │
├─────────────────────────┼────────────────────────┤
│  Server Components      │  Edge Runtime          │
│  (streaming, RSC)       │  (limited APIs)        │
├─────────────────────────┴────────────────────────┤
│  API Routes (Node.js runtime)                    │
│  (full Node, different from Edge)                │
└─────────────────────────────────────────────────┘
```

Now ask yourself: which runtime is your code running in? Are you sure?

**PeARL's answer:**

```
┌─────────────────────────────────────────────────┐
│  Browser: React components                       │
│  (just rendering, no business logic)             │
├─────────────────────────────────────────────────┤
│  Server: Elixir (Phoenix + LiveView + Ash)       │
│  (one runtime, one language, all logic here)    │
└─────────────────────────────────────────────────┘
```

---

## The Vercel + Supabase Trap

These tools are optimized for **starting**, not **sustaining**.

### Vercel

- **Cold starts** hurt latency (especially on free/hobby tiers)
- **Bandwidth charges** can spike unexpectedly ($500 surprise bills are common)
- **Edge runtime limitations** mean you can't use all Node.js APIs
- **Vendor lock-in** is significant (try migrating away)

### Supabase

- **Row limits** force tier upgrades at awkward times
- **Realtime is expensive** at scale
- **Auth quirks** (JWT expiration, session management)
- **RLS policies** are powerful but error-prone (easy to accidentally expose data)

### Fly.io (PeARL's default)

- **Predictable pricing** — you pay for VMs, not invocations
- **No cold starts** — your app is always running
- **Simple mental model** — it's a VM, not magic
- **Easy to migrate** — no proprietary lock-in

---

## "Isn't Elixir Harder to Learn?"

Harder to **start**. Easier to **understand**.

**JavaScript looks familiar** but you're constantly debugging:
- Why did this re-render?
- Why is this state stale?
- Why does `this` point to undefined?
- Why does this work locally but not in production?

**Elixir looks unfamiliar** but you can read it:

```elixir
# Even if you don't know Elixir, you can read this:
def complete_order(order) do
  order
  |> validate_payment()
  |> charge_card()
  |> send_confirmation()
  |> update_status(:completed)
end
```

Data flows top to bottom. No hidden state. No side effects surprises.

---

## The Ecosystem Concern

"But npm has a package for everything!"

True. Consider:
- How many of those packages are maintained?
- How many have security vulnerabilities?
- How many will break on the next Node.js version?
- How many will break on the next React version?

**PeARL's approach:** Fewer dependencies, each one battle-tested.

- Phoenix has been in production since 2014
- The BEAM has been in production since 1986
- PostgreSQL has been in production since 1996
- React has been in production since 2013

We're not using bleeding-edge tech. We're using **boring technology that works**.

---

## When Next.js Is Right

To be fair, Next.js is a good choice when:

- Your team already knows it deeply (not just "knows JavaScript")
- You're building a content-heavy marketing site
- You need SSG for SEO-critical pages
- You're integrating with a Vercel-optimized stack (headless CMS, etc.)

PeARL is not the right choice for everyone. It's the right choice for:

- Beginners who want guardrails
- Teams who value readability over cleverness
- Apps that need real-time features
- Founders who want to understand their own code

---

## The Real Comparison

| Dimension | Next.js + Supabase | PeARL |
|---|---|---|
| **Learning curve shape** | Gentle start, constant spikes | Steeper start, then smooth |
| **State management** | You figure it out (Redux? Zustand? Context?) | Server owns it (LiveView) |
| **Real-time** | Bolt it on (Supabase Realtime, separate concern) | Built in (LiveView, same model) |
| **Authorization** | Add it yourself (easy to forget) | Deny by default (Ash) |
| **Deployment** | Vercel + Supabase (two bills, two vendors) | Fly.io (one bill, no lock-in) |
| **AI assistance** | Good React generation | Good React generation + readable Elixir |
| **Debugging** | "Which of three runtimes is this?" | "It's on the server" |
| **6-month maintainability** | "What does this useEffect do again?" | Read the DSL |

---

## Make Your Choice

Neither stack is universally "better." They optimize for different things.

**Next.js optimizes for:** Familiar syntax, fast starts, marketing sites, content.

**PeARL optimizes for:** Beginner completion rate, long-term readability, real-time apps, maintainability.

If you've been frustrated by the JS ecosystem's footguns, PeARL might be worth the initial learning curve.

---

## Next Steps

- [Architecture](./architecture) — See how PeARL layers work
- [Getting Started](./getting-started) — Try it yourself
