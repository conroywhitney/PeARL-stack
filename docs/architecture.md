---
layout: page
title: PeARL Architecture
permalink: /architecture/
nav_order: 3
---

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

  # mount/3 runs once when user connects
  # This is your "page load" — but over WebSocket, not HTTP
  def mount(_params, _session, socket) do
    # FOOTGUN AVOIDED: We pass `actor:` to every Ash call
    # This isn't optional — Ash policies will REJECT the call without it
    # In JS-land, forgetting auth checks silently exposes data
    todos = MyApp.Todos.list_todos!(actor: socket.assigns.current_user)

    # assign/3 stores data in the socket
    # When this changes, live_react auto-updates your React props
    {:ok, assign(socket, :todos, todos)}
  end

  # handle_event/3 receives events from React's pushEvent()
  # The string "complete_todo" matches what React sends
  # The map %{"id" => id} pattern-matches the payload
  def handle_event("complete_todo", %{"id" => id}, socket) do
    # FOOTGUN AVOIDED: We fetch the todo WITH the actor
    # Ash policies ensure users can only access their own todos
    # No "IDOR vulnerability" possible — it's enforced at the data layer
    todo = MyApp.Todos.get_todo!(id, actor: socket.assigns.current_user)

    # FOOTGUN AVOIDED: The action name :complete is explicit
    # You can't accidentally call the wrong action or pass wrong fields
    # Ash validates everything before touching the database
    MyApp.Todos.complete_todo!(todo, actor: socket.assigns.current_user)

    # Re-fetch the list and update assigns
    # live_react sees the change and pushes new props to React
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

  # ATTRIBUTES: What fields exist on this resource
  # This is your "schema" — but with built-in validations
  attributes do
    uuid_primary_key :id

    # FOOTGUN AVOIDED: `allow_nil?: false` means the database AND Ash
    # will reject empty titles. No "undefined" sneaking through.
    attribute :title, :string, allow_nil?: false

    attribute :completed_at, :utc_datetime

    # timestamps() adds inserted_at and updated_at automatically
    # No forgetting to add these, no inconsistent naming
    timestamps()
  end

  # ACTIONS: The ONLY ways to interact with this resource
  # There's no "raw SQL" backdoor. Every operation goes through here.
  actions do
    # defaults [:read, :destroy] gives you basic list/get/delete
    defaults [:read, :destroy]

    # FOOTGUN AVOIDED: `accept [:title]` is an allowlist
    # Even if someone sends {title: "x", is_admin: true},
    # the is_admin field is IGNORED — not "mass assigned"
    # Rails devs know this pain. Ash prevents it by default.
    create :create do
      accept [:title]
      # Automatically associate this todo with whoever created it
      change relate_actor(:user)
    end

    # FOOTGUN AVOIDED: This action ONLY sets completed_at
    # You can't accidentally modify other fields through this action
    # The behavior is explicit and auditable
    update :complete do
      change set_attribute(:completed_at, &DateTime.utc_now/0)
    end
  end

  # POLICIES: Who can do what — DENY BY DEFAULT
  # If no policy matches, the action is REJECTED
  # Forget to add a policy? Action fails. Data stays safe.
  policies do
    # FOOTGUN AVOIDED: Users can only read their OWN todos
    # This isn't a "reminder to check" — it's ENFORCED
    # No IDOR vulnerabilities possible
    policy action_type(:read) do
      authorize_if relates_to_actor_via(:user)
    end

    # Same for completing — only the owner can complete their todo
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
