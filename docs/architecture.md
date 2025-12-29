---
layout: page
title: Architecture
permalink: /architecture/
nav_order: 3
---

# PeARL Architecture

Four layers. Unidirectional dependencies. No spaghetti.

---

## The Four Layers

```
┌─────────────────────────────────────────────────────────────┐
│                     REACT COMPONENTS                         │
│                                                              │
│   • Pure UI rendering                                        │
│   • Receives props from LiveView (auto-updates!)             │
│   • Sends events via pushEvent()                             │
│   • No business logic                                        │
│                                                              │
├──────────────────────────┬──────────────────────────────────┤
│        pushEvent()       │       props auto-update           │
│            ↓             │              ↑                    │
├──────────────────────────┴──────────────────────────────────┤
│                         LIVEVIEW                             │
│                                                              │
│   • Owns server-side state                                   │
│   • Handles events from React                                │
│   • Calls Ash actions                                        │
│   • Updates assigns → React auto-rerenders                   │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                       ASH DOMAINS                            │
│                                                              │
│   • Business logic & validations                             │
│   • Authorization policies (deny by default)                 │
│   • Data transformations                                     │
│   • Database operations                                      │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                       POSTGRESQL                             │
│                                                              │
│   • Persistence                                              │
│   • Constraints                                              │
│   • Transactions                                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## The Rule

**Each layer can only call down.**

- React calls LiveView (via events)
- LiveView calls Ash
- Ash calls PostgreSQL

Never the other way around. This creates:
- **Clear data flow** — always know where data comes from
- **Easy debugging** — trace down, never up
- **Testable layers** — each layer can be tested independently

---

## Layer 1: React Components

React does what React does best: render UI.

```jsx
function TodoList({ todos, pushEvent }) {
  const handleComplete = (id) => {
    pushEvent("complete_todo", { id })
  }

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          {todo.title}
          <button onClick={() => handleComplete(todo.id)}>
            Done
          </button>
        </li>
      ))}
    </ul>
  )
}
```

**What React does:**
- Receives data as props
- Renders UI
- Sends events up

**What React doesn't do:**
- Manage global state
- Make API calls
- Handle authentication
- Run business logic

---

## Layer 2: LiveView

LiveView coordinates everything.

```elixir
defmodule MyAppWeb.TodoLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    todos = MyApp.Todos.list_todos!(actor: socket.assigns.current_user)
    {:ok, assign(socket, :todos, todos)}
  end

  def handle_event("complete_todo", %{"id" => id}, socket) do
    todo = MyApp.Todos.get_todo!(id, actor: socket.assigns.current_user)
    MyApp.Todos.complete_todo!(todo, actor: socket.assigns.current_user)

    # Re-fetch and push updated list
    todos = MyApp.Todos.list_todos!(actor: socket.assigns.current_user)
    {:noreply, assign(socket, :todos, todos)}
  end
end
```

**What LiveView does:**
- Maintains WebSocket connection
- Owns server-side state
- Handles events from React
- Calls Ash actions (with actor for authorization)
- Updates assigns → `live_react` auto-updates React props

**What LiveView doesn't do:**
- Contain business logic
- Validate data
- Query the database directly

---

## Layer 3: Ash Domains

Ash is where your business lives.

```elixir
defmodule MyApp.Todos.Todo do
  use Ash.Resource,
    domain: MyApp.Todos,
    data_layer: AshPostgres.DataLayer

  attributes do
    uuid_primary_key :id
    attribute :title, :string, allow_nil?: false
    attribute :completed_at, :utc_datetime
    timestamps()
  end

  actions do
    defaults [:read, :destroy]

    create :create do
      accept [:title]
      change relate_actor(:user)
    end

    update :complete do
      change set_attribute(:completed_at, &DateTime.utc_now/0)
    end
  end

  policies do
    # Deny by default — must explicitly authorize
    policy action_type(:read) do
      authorize_if relates_to_actor_via(:user)
    end

    policy action(:complete) do
      authorize_if relates_to_actor_via(:user)
    end
  end

  relationships do
    belongs_to :user, MyApp.Accounts.User
  end
end
```

**What Ash does:**
- Defines your data model
- Validates all input
- Enforces authorization
- Runs database operations

**What Ash doesn't do:**
- Handle HTTP
- Manage WebSocket state
- Render UI

---

## Layer 4: PostgreSQL

The database is just storage. All logic lives above.

```sql
-- Ash generates migrations for you
CREATE TABLE todos (
  id UUID PRIMARY KEY,
  title VARCHAR NOT NULL,
  completed_at TIMESTAMP,
  user_id UUID REFERENCES users(id),
  inserted_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

---

## The Event Flow

Here's a complete journey through all four layers:

```
User clicks "Complete"
        │
        ▼
┌─────────────────────────────────────────┐
│ React: pushEvent("complete_todo", {id}) │
└─────────────────────────────────────────┘
        │
        │ WebSocket
        ▼
┌─────────────────────────────────────────┐
│ LiveView: handle_event("complete_todo") │
│   → calls Todos.complete_todo!(todo)    │
└─────────────────────────────────────────┘
        │
        │ Function call
        ▼
┌─────────────────────────────────────────┐
│ Ash: complete action                    │
│   → checks policy (user owns todo?)     │
│   → sets completed_at                   │
│   → writes to database                  │
└─────────────────────────────────────────┘
        │
        │ SQL
        ▼
┌─────────────────────────────────────────┐
│ PostgreSQL: UPDATE todos SET ...        │
└─────────────────────────────────────────┘
        │
        │ Response bubbles up
        ▼
┌─────────────────────────────────────────┐
│ LiveView: assign(socket, :todos, todos) │
│   → live_react detects change           │
└─────────────────────────────────────────┘
        │
        │ Props auto-update via WebSocket
        ▼
┌─────────────────────────────────────────┐
│ React: receives new todos as props      │
│   → re-renders automatically            │
└─────────────────────────────────────────┘
        │
        ▼
User sees completed todo
```

**The magic:** You don't write `handleEvent` in React for data updates. When LiveView assigns change, `live_react` automatically sends the new props to your component. React just re-renders with fresh data.

---

## Why This Works

### 1. No Hidden State

State lives in exactly one place: LiveView. React renders it. Ash transforms it. PostgreSQL stores it.

### 2. Authorization Can't Be Bypassed

Every Ash action requires an `actor`. Policies are checked automatically. Forget to pass the actor? The call fails. There's no "oops, I exposed data" path.

### 3. Debugging Is Linear

Something wrong? Trace down:
1. Did React send the right event?
2. Did LiveView handle it correctly?
3. Did Ash accept the input?
4. Did PostgreSQL get the right data?

### 4. Changes Are Contained

Need to change how "complete" works? Change the Ash action. LiveView and React don't care about the implementation.

---

## Next Steps

- [Why PeARL?](./why) — The philosophy behind these choices
- [vs. Next.js](./vs-nextjs) — How this compares
- [Getting Started](./getting-started) — Build something
