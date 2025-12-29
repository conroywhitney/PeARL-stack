---
layout: page
title: Why PeARL?
permalink: /why/
nav_order: 2
---

# Why PeARL?

Every technology choice in PeARL was made to answer one question:

> **What helps beginners finish their apps?**

Not "what's trending." Not "what has the most npm downloads." Not "what looks impressive on a resume."

What actually works.

---

## Why Phoenix?

Phoenix is a web framework built on the BEAM—the Erlang virtual machine that powers WhatsApp, Discord, and telecom systems that literally cannot go down.

**What you get:**
- WebSockets are first-class, not bolted on
- Real-time by default, not as an afterthought
- One language (Elixir) for the entire backend
- Battle-tested reliability from 30+ years of Erlang

**The key insight:** Phoenix was designed for persistent connections. The BEAM was literally built to handle millions of concurrent users. This isn't a marketing claim—it's what telecom systems have relied on for decades.

---

## Why Events?

Most stacks teach you to think in terms of REST APIs:
- `GET /api/todos`
- `POST /api/todos`
- `PUT /api/todos/:id`
- `DELETE /api/todos/:id`

Then you need: authentication headers, CORS configuration, API versioning, error handling, caching strategies, rate limiting...

**PeARL's approach:** Events in, events out.

```javascript
// That's it. That's the whole contract.
pushEvent("create_todo", { title: "Buy milk" })
```

The server handles it. The server pushes back what changed. Your UI updates.

**No:**
- REST endpoints to design
- GraphQL schemas to maintain
- CORS to debug
- API versions to manage
- HTTP caching to invalidate
- "Why is my data stale?" mysteries

Just one WebSocket. Events in. Events out.

---

## Why Ash?

Ash is a declarative framework for modeling your domain—users, posts, comments, orders, whatever your app needs.

**The killer feature: Deny by default.**

```elixir
policies do
  # Nothing is allowed until you explicitly permit it
  policy action_type(:read) do
    authorize_if relates_to_actor_via(:user)
  end
end
```

In most frameworks, if you forget to add authorization, the action works (and exposes data). In Ash, if you forget to add authorization, the action fails. You must explicitly permit access.

**What else Ash gives you:**
- All your domain logic in one readable place
- Automatic validations that can't be bypassed
- Built-in pagination, filtering, sorting
- No raw SQL (unless you want it)
- GraphQL and JSON:API for free (when you need them)

**The readable DSL:**

```elixir
actions do
  create :register do
    accept [:email, :name]
    change set_attribute(:role, :member)
  end
end
```

Even if you don't know Elixir, you can read what this does.

---

## Why React?

React in PeARL is React as originally intended: **a reactive UI library.**

Not React + Redux + React Query + Zustand + Context + custom hooks managing seventeen different pieces of state.

Just React. Rendering things. Responding to events.

**Why keep React at all?**
- Massive ecosystem of components
- AI generates excellent React code (more training data)
- Path to React Native for mobile
- Familiar to most web developers
- Vibrant community and tooling

**The key:** LiveView owns the state. React just renders it.

```javascript
// LiveView tells you what to show
handleEvent("todos_updated", ({ todos }) => {
  setTodos(todos)  // Just render it
})

// You tell LiveView what happened
<button onClick={() => pushEvent("complete_todo", { id: todo.id })}>
  Done
</button>
```

No useEffect loops. No stale closures. No "why did this re-render 47 times?"

---

## Why LiveView?

LiveView is the coordination layer. It maintains a persistent WebSocket connection and owns server-side state.

**What this means:**
- Real-time updates without polling
- Progressive re-renders (only what changed)
- Server-side state = no sync bugs
- Automatic cleanup when users disconnect

**The mental model:**

```
Browser                    Server
   │                          │
   │──pushEvent("add_item")──▶│
   │                          │ (runs Ash action)
   │                          │ (updates state)
   │◀──handleEvent("updated")─│
   │                          │
   └──────────────────────────┘
        One WebSocket
```

Compare this to the JS approach:
- REST call
- Wait for response
- Update local state
- Hope it matches server state
- Poll for updates?
- WebSocket for real-time?
- Now you have two sources of truth...

---

## The Frustration Curve

```
Frustration
    │
    │     ┌─────┐
    │     │     │     Next.js + Supabase
    │  ┌──┤     ├──┐     ┌───┐
    │  │  │     │  │  ┌──┤   ├──┐
    │  │  │     │  │  │  │   │  │  ...constant spikes
    │──┴──┴─────┴──┴──┴──┴───┴──┴──────────────────▶
    │
    │  ┌────┐
    │  │    │         PeARL
    │  │    └─────────────────────────────────────▶
    │──┴───────────────────────────────────────────
    │     │
    │     └─ "what's |> and %{}?"
    │
    └──────────────────────────────────────────────▶ Time
```

**The JS path** seems flatter at first—JavaScript is familiar! But you'll hit constant frustration spikes:
- useEffect infinite loops
- Hydration mismatches
- Stale closures
- "use client" vs "use server" confusion
- Surprise Vercel bills

**The PeARL path** has one initial learning curve (Elixir syntax), then levels out. The language and framework were designed to be predictable.

---

## The Philosophy

### Constraints = Freedom

When you can't accidentally:
- Expose sensitive fields in an API response
- Bypass authorization on an action
- Create invalid state in your database
- Forget to clean up a subscription

...you spend your time building features, not debugging edge cases.

### Explicit Over Magic

Every piece of code in PeARL does what it looks like it does:

```elixir
# This reads exactly like what it does
policies do
  policy action(:delete) do
    authorize_if actor_attribute_equals(:role, :admin)
  end
end
```

No hidden state. No surprise re-renders. No "it works in dev but breaks in prod."

### Readable by Humans

The goal isn't clever code. The goal is code you can read six months later—or hand to a new team member—and have them understand it in minutes.

---

## Next Steps

- [Architecture](./architecture) — See how the layers work together
- [vs. Next.js](./vs-nextjs) — Detailed comparison
- [Getting Started](./getting-started) — Start building
