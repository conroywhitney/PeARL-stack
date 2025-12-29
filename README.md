# PeARL Stack

**P**hoenix · **e**vents · **A**sh · **R**eact · **L**iveView

The beginner-friendly, vibe-proof stack.

**[Read the docs →](https://conroywhitney.github.io/PeARL-stack/)**

---

## What is this?

PeARL is an opinionated web stack that locks away the footguns and maximizes readability.

- **Events in, events out** — No REST, no GraphQL, just WebSocket events
- **Deny by default** — Ash policies prevent accidental data exposure
- **Server-owned state** — LiveView manages state, React just renders
- **Four clean layers** — Each layer only calls down, never up

## Quick Links

- [Why PeARL?](https://conroywhitney.github.io/PeARL-stack/why/) — The philosophy
- [Architecture](https://conroywhitney.github.io/PeARL-stack/architecture/) — How it works
- [vs. Next.js](https://conroywhitney.github.io/PeARL-stack/vs-nextjs/) — Honest comparison
- [Getting Started](https://conroywhitney.github.io/PeARL-stack/getting-started/) — Build something

## The Stack

| Layer | Technology | Role |
|-------|------------|------|
| **P** | Phoenix | Web framework on the battle-tested BEAM |
| **e** | events | The entire API: `pushEvent` / `handleEvent` |
| **A** | Ash | Declarative domains with deny-by-default auth |
| **R** | React | UI rendering (not state management) |
| **L** | LiveView | WebSocket state coordination |

## License

MIT
