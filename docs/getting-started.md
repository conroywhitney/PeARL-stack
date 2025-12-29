---
layout: page
title: Getting Started
permalink: /getting-started/
nav_order: 5
---

# Getting Started with PeARL

Ready to build something? Here's how to start.

---

## Prerequisites

You'll need:

- **Elixir 1.15+** — [Installation guide](https://elixir-lang.org/install.html)
- **PostgreSQL 14+** — [Download](https://www.postgresql.org/download/)
- **Node.js 18+** — [Download](https://nodejs.org/)

Verify your setup:

```bash
elixir --version   # Should show 1.15 or higher
psql --version     # Should show 14 or higher
node --version     # Should show 18 or higher
```

---

## Quick Start

### 1. Create a new Phoenix project with Ash

```bash
mix igniter.new my_app \
  --install ash,ash_postgres,ash_authentication,ash_phoenix \
  --with phx.new
```

### 2. Add live_react for React integration

```bash
cd my_app
mix igniter.install live_react
```

### 3. Start the server

```bash
mix setup
mix phx.server
```

Visit [http://localhost:4000](http://localhost:4000) and you're running!

---

## Project Structure

```
my_app/
├── lib/
│   ├── my_app/                    # Ash domains (business logic)
│   │   ├── accounts/              # Users, auth
│   │   │   ├── accounts.ex        # Domain definition
│   │   │   └── user.ex            # User resource
│   │   └── todos/                 # Your domain
│   │       ├── todos.ex           # Domain definition
│   │       └── todo.ex            # Todo resource
│   │
│   └── my_app_web/                # Phoenix web layer
│       ├── live/                  # LiveView modules
│       │   └── todo_live.ex       # Handles todo events
│       ├── components/            # HEEx components (optional)
│       └── router.ex              # Routes
│
├── assets/
│   └── react-components/          # React components
│       └── TodoList.jsx           # Your React UI
│
└── priv/
    └── repo/
        └── migrations/            # Database migrations
```

---

## Your First Feature: A Todo List

### Step 1: Define the Ash Resource

```elixir
# lib/my_app/todos/todo.ex
defmodule MyApp.Todos.Todo do
  use Ash.Resource,
    domain: MyApp.Todos,
    data_layer: AshPostgres.DataLayer

  postgres do
    table "todos"
    repo MyApp.Repo
  end

  attributes do
    uuid_primary_key :id

    # allow_nil?: false = this field is REQUIRED
    # Ash will reject creates/updates with empty titles
    # No "undefined" or "" sneaking into your database
    attribute :title, :string, allow_nil?: false

    # default: false = new todos start incomplete
    # This is set at the Ash layer, not the database
    # So it works even if you forget the DB default
    attribute :completed, :boolean, default: false

    # Adds inserted_at and updated_at automatically
    timestamps()
  end

  actions do
    # defaults [:read, :destroy] gives you:
    # - list all todos
    # - get a single todo by id
    # - delete a todo
    defaults [:read, :destroy]

    create :create do
      # FOOTGUN AVOIDED: `accept` is an ALLOWLIST
      # Only :title can be set on create
      # Even if someone sends {title: "x", completed: true},
      # the completed field is IGNORED
      # No "mass assignment" vulnerabilities
      accept [:title]
    end

    update :complete do
      # This action does ONE thing: mark as completed
      # You can't pass arbitrary fields to modify
      # The behavior is explicit and predictable
      change set_attribute(:completed, true)
    end
  end

  # POLICIES: Who can do what
  # Start open for learning, then tighten for production!
  #
  # IMPORTANT: In production, you'd write:
  #   policy action_type(:read) do
  #     authorize_if relates_to_actor_via(:user)
  #   end
  # This ensures users only see THEIR todos (no IDOR bugs)
  policies do
    policy always() do
      authorize_if always()
    end
  end
end
```

### Step 2: Create the Domain

```elixir
# lib/my_app/todos/todos.ex
defmodule MyApp.Todos do
  use Ash.Domain

  resources do
    resource MyApp.Todos.Todo
  end
end
```

### Step 3: Generate the Migration

```bash
mix ash.codegen create_todos
mix ash.migrate
```

### Step 4: Create the LiveView

```elixir
# lib/my_app_web/live/todo_live.ex
defmodule MyAppWeb.TodoLive do
  use MyAppWeb, :live_view

  # mount/3 runs when a user visits this page
  # It's like "componentDidMount" but on the server
  def mount(_params, _session, socket) do
    # Fetch all todos from the database via Ash
    # The ! means "raise an error if this fails" (fail fast, don't hide bugs)
    todos = MyApp.Todos.Todo |> Ash.read!()

    # assign/3 stores data in the socket's state
    # This becomes @todos in your template
    {:ok, assign(socket, :todos, todos)}
  end

  # render/1 returns the HTML template
  # The ~H sigil is "HEEx" — HTML + Elixir
  def render(assigns) do
    ~H"""
    <.react
      name="TodoList"
      todos={@todos}
    />
    """
    # ↑ That's it! @todos flows to React as props
    # When @todos changes, React automatically re-renders
    # No handleEvent needed on the React side
  end

  # handle_event/3 receives events from React's pushEvent()
  # "create_todo" matches pushEvent("create_todo", {...})
  # %{"title" => title} pattern-matches the payload and extracts `title`
  def handle_event("create_todo", %{"title" => title}, socket) do
    # Create a new todo via Ash
    # for_create/3 builds a "changeset" (a pending change)
    # Ash.create!/1 executes it and saves to database
    MyApp.Todos.Todo
    |> Ash.Changeset.for_create(:create, %{title: title})
    |> Ash.create!()

    # Re-fetch all todos and update assigns
    # live_react will automatically push new props to React
    todos = MyApp.Todos.Todo |> Ash.read!()
    {:noreply, assign(socket, :todos, todos)}
    # ↑ :noreply means "don't send a separate reply"
    # The assign change IS the reply (via live_react)
  end

  def handle_event("complete_todo", %{"id" => id}, socket) do
    # Fetch the specific todo by ID
    MyApp.Todos.Todo
    |> Ash.get!(id)
    # Build an update changeset using the :complete action
    # Remember: :complete can ONLY set completed=true (by design!)
    |> Ash.Changeset.for_update(:complete, %{})
    |> Ash.update!()

    # Re-fetch and update — React re-renders automatically
    todos = MyApp.Todos.Todo |> Ash.read!()
    {:noreply, assign(socket, :todos, todos)}
  end
end
```

### Step 5: Create the React Component

```jsx
// assets/react-components/TodoList.jsx
import React, { useState } from 'react'

// NOTICE: No useEffect! No data fetching! No global state!
//
// `todos` comes from LiveView as a prop (automatically updated)
// `pushEvent` is injected by live_react to send events to the server
//
// This is React as it was meant to be: a UI rendering library
export default function TodoList({ todos, pushEvent }) {
  // Local state is fine for UI-only concerns (like form input)
  const [newTitle, setNewTitle] = useState('')

  const handleSubmit = (e) => {
    e.preventDefault()
    if (newTitle.trim()) {
      // pushEvent sends to LiveView's handle_event/3
      // LiveView will create the todo and update assigns
      // Then live_react pushes new `todos` prop back to us
      // We re-render automatically — no manual state management!
      pushEvent('create_todo', { title: newTitle })
      setNewTitle('')
    }
  }

  const handleComplete = (id) => {
    // Same pattern: tell the server what happened
    // Server updates database, assigns change, we re-render
    pushEvent('complete_todo', { id })
  }

  // Pure rendering — just show what we're given
  return (
    <div className="max-w-md mx-auto p-4">
      <h1 className="text-2xl font-bold mb-4">My Todos</h1>

      <form onSubmit={handleSubmit} className="mb-4">
        <input
          type="text"
          value={newTitle}
          onChange={(e) => setNewTitle(e.target.value)}
          placeholder="What needs to be done?"
          className="border p-2 w-full rounded"
        />
        <button
          type="submit"
          className="mt-2 bg-blue-500 text-white px-4 py-2 rounded"
        >
          Add Todo
        </button>
      </form>

      {/* todos prop comes from LiveView — always fresh, always in sync */}
      <ul className="space-y-2">
        {todos.map(todo => (
          <li
            key={todo.id}
            className={`p-2 border rounded flex justify-between ${
              todo.completed ? 'bg-green-50 line-through' : ''
            }`}
          >
            <span>{todo.title}</span>
            {!todo.completed && (
              <button
                onClick={() => handleComplete(todo.id)}
                className="text-green-600 hover:text-green-800"
              >
                Complete
              </button>
            )}
          </li>
        ))}
      </ul>
    </div>
  )
}
```

### Step 6: Add the Route

```elixir
# lib/my_app_web/router.ex
scope "/", MyAppWeb do
  pipe_through :browser

  live "/todos", TodoLive
end
```

### Step 7: Run It

```bash
mix phx.server
```

Visit [http://localhost:4000/todos](http://localhost:4000/todos) and you have a working todo app!

---

## What You Just Built

In ~100 lines of code:

- **Database-backed todos** with timestamps
- **Real-time updates** via WebSocket
- **React UI** with familiar patterns
- **Server-owned state** (no Redux, no Zustand)
- **Type-safe actions** (create, complete)
- **Ready for authorization** (just tighten the policies)

---

## Next Steps

### Add Authorization

Replace the open policies with real ones:

```elixir
policies do
  policy action_type(:read) do
    authorize_if relates_to_actor_via(:user)
  end

  policy action(:create) do
    authorize_if actor_present()
  end
end
```

### Add Authentication

```bash
mix igniter.install ash_authentication_phoenix
```

### Deploy to Fly.io

```bash
fly launch
fly deploy
```

---

## Learning Resources

- [Phoenix Framework Guides](https://hexdocs.pm/phoenix/overview.html)
- [Ash Framework Documentation](https://hexdocs.pm/ash/get-started.html)
- [LiveView Documentation](https://hexdocs.pm/phoenix_live_view)
- [live_react Documentation](https://hexdocs.pm/live_react)

---

## Getting Help

Stuck? Here's where to find help:

- [Elixir Forum](https://elixirforum.com/) — Friendly, active community
- [Ash Discord](https://discord.gg/D7FNG2q) — Direct help from Ash maintainers
- [Phoenix Slack](https://elixir-slackin.herokuapp.com/) — Real-time chat

---

<p style="text-align: center; margin-top: 3rem;">
<strong>Welcome to PeARL.</strong><br/>
Build something real. Ship it. Understand it.
</p>
