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
    attribute :title, :string, allow_nil?: false
    attribute :completed, :boolean, default: false
    timestamps()
  end

  actions do
    defaults [:read, :destroy]

    create :create do
      accept [:title]
    end

    update :complete do
      change set_attribute(:completed, true)
    end
  end

  # Start with open policies for learning
  # (Tighten these before production!)
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

  def mount(_params, _session, socket) do
    todos = MyApp.Todos.Todo |> Ash.read!()
    {:ok, assign(socket, :todos, todos)}
  end

  def render(assigns) do
    ~H"""
    <.react
      name="TodoList"
      todos={@todos}
    />
    """
  end

  def handle_event("create_todo", %{"title" => title}, socket) do
    MyApp.Todos.Todo
    |> Ash.Changeset.for_create(:create, %{title: title})
    |> Ash.create!()

    todos = MyApp.Todos.Todo |> Ash.read!()
    {:noreply, assign(socket, :todos, todos)}
  end

  def handle_event("complete_todo", %{"id" => id}, socket) do
    MyApp.Todos.Todo
    |> Ash.get!(id)
    |> Ash.Changeset.for_update(:complete, %{})
    |> Ash.update!()

    todos = MyApp.Todos.Todo |> Ash.read!()
    {:noreply, assign(socket, :todos, todos)}
  end
end
```

### Step 5: Create the React Component

```jsx
// assets/react-components/TodoList.jsx
import React, { useState } from 'react'

export default function TodoList({ todos, pushEvent }) {
  const [newTitle, setNewTitle] = useState('')

  const handleSubmit = (e) => {
    e.preventDefault()
    if (newTitle.trim()) {
      pushEvent('create_todo', { title: newTitle })
      setNewTitle('')
    }
  }

  const handleComplete = (id) => {
    pushEvent('complete_todo', { id })
  }

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
