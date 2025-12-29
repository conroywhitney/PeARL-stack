---
layout: home
title: PeARL Stack
nav_order: 1
---

**P**hoenix · **e**vents · **A**sh · **R**eact · **L**iveView

The beginner-friendly, vibe-proof stack where the footguns have been locked away.

---

## What is PeARL?

PeARL is an opinionated web application stack designed for **maximum readability and maintainability**. It combines battle-tested technologies in a way that prevents common mistakes rather than just documenting them.

```
┌─────────────────────────────────────────┐
│           React Components              │  ← UI you can see
├─────────────────────────────────────────┤
│              LiveView                   │  ← Real-time state
├─────────────────────────────────────────┤
│            Ash Domains                  │  ← Business logic
├─────────────────────────────────────────┤
│            PostgreSQL                   │  ← Your data
└─────────────────────────────────────────┘
```

Each layer can only call down. No spaghetti. No mystery re-renders. No "works on my machine."

---

## The Stack

| Letter | Technology | Role |
|--------|------------|------|
| **P** | [Phoenix](https://phoenixframework.org/) | Web framework built for real-time, on the battle-tested BEAM |
| **e** | events | The entire API: `pushEvent` down, props auto-update up |
| **A** | [Ash](https://ash-hq.org/) | Declarative domain modeling with deny-by-default permissions |
| **R** | [React](https://react.dev/) | UI components, as originally intended—reactive, not state-managing |
| **L** | [LiveView](https://hexdocs.pm/phoenix_live_view/) | WebSocket state coordination, progressive re-renders |

---

## Why "events"?

The lowercase **e** is intentional. Events aren't a proper noun—they're the philosophy.

Your entire client-server contract is:

```jsx
// React → Server (you send events)
pushEvent("create_todo", { title: "Buy milk" })

// Server → React (props auto-update)
<.react name="TodoList" todos={@todos} />
// When @todos changes, your component re-renders automatically
```

No REST. No GraphQL. No CORS. No API versioning. No caching bugs.

Just `pushEvent` up, props down.

---

## The Thesis

> Non-technical founders complete their apps more often with PeARL than with JavaScript stacks—even though JS "seems more familiar."

**Why?**
- Fewer footguns = less frustration = more founders finish
- Explicit over magic = better understanding = confident iteration
- Constraints = freedom (can't accidentally bypass auth or expose sensitive data)
- Server-owned state = simpler mental model = less debugging

---

## Who is this for?

PeARL is designed for:

- **AI-assisted developers** who generated something that works but don't fully understand it
- **Non-technical founders** building their first real product
- **Experienced devs** tired of debugging useEffect loops and hydration errors
- **Teams** who want readable code that new members can understand in days, not months

---

## Quick Links

- [Why PeARL?](./why) — The philosophy and reasoning
- [Architecture](./architecture) — How the layers work together
- [vs. Next.js](./vs-nextjs) — Honest comparison with the JS path
- [Getting Started](./getting-started) — Start building

---

<p style="text-align: center; color: #666; font-size: 0.9em;">
Built with technologies that have been running in production for decades.<br/>
Phoenix. Elixir. The BEAM. PostgreSQL.<br/>
<em>Boring technology. Exciting results.</em>
</p>
