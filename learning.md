# Learning Guide — Todo App (Rails 8 + PostgreSQL)

This walks through how to start this project day-to-day, and how the pieces fit together — using the real files in this app as examples.

## 1. How to Start the Project

### Every time you come back to work on it

```
cd "D:\development\office\Live after 10-07-2026\ROR proejct\todo_app"
bin/rails server -p 3000
```

Then open http://localhost:3000 in a browser.

If `ruby`, `rails`, or `psql` aren't recognized in a terminal, either open a **new** terminal (PATH was updated permanently, existing terminals opened before the install won't see it) or run:
```
$env:Path += ";C:\Ruby33-x64\bin;C:\Program Files\PostgreSQL\17\bin"   # PowerShell
```

### First time on a new machine / after cloning

```
bundle install          # install gems from Gemfile
bin/rails db:prepare    # create databases + run migrations
bin/rails server -p 3000
```

### Checking the database is really running

PostgreSQL runs as a Windows service (`postgresql-x64-17`) — it starts automatically on boot. If the app can't connect:
```
Get-Service postgresql-x64-17     # should say "Running"
```

### Stopping the server

`Ctrl+C` in the terminal running `bin/rails server`.

## 2. The Big Picture: MVC

Rails is built around **MVC (Model–View–Controller)**, plus routing that decides which controller handles each URL. A request flows like this:

```
Browser request  →  routes.rb  →  Controller  →  Model  →  Database
                                       ↓
                                     View  →  HTML response
```

Everything in this app follows that same flow for the `Todo` resource. Below is each piece, in the order a request actually travels through them.

## 3. Routing — `config/routes.rb`

```ruby
resources :todos
root "todos#index"
```

`resources :todos` is one line that generates 7 routes (the standard CRUD set):

| HTTP verb | URL | Controller#action | Purpose |
|---|---|---|---|
| GET | /todos | todos#index | list all todos |
| GET | /todos/new | todos#new | show the "new todo" form |
| POST | /todos | todos#create | save a new todo |
| GET | /todos/:id | todos#show | show one todo |
| GET | /todos/:id/edit | todos#edit | show the edit form |
| PATCH/PUT | /todos/:id | todos#update | save changes |
| DELETE | /todos/:id | todos#destroy | delete a todo |

`root "todos#index"` just means `/` shows the same page as `/todos`.

Run `bin/rails routes | grep todo` any time to see this table generated live from your code.

## 4. Model — `app/models/todo.rb`

```ruby
class Todo < ApplicationRecord
end
```

This looks empty, but it's not — by inheriting from `ApplicationRecord` (Rails' Active Record base class), `Todo` automatically:
- maps to the `todos` table in PostgreSQL
- gets a Ruby attribute for every column (`title`, `description`, `completed`, `created_at`, `updated_at`)
- gets methods like `Todo.all`, `Todo.find(1)`, `.save`, `.update`, `.destroy` for free

This is **convention over configuration**: the model class name (singular `Todo`) maps to the table name (plural `todos`) automatically. No manual SQL, no manual mapping.

### Where the columns come from — `db/migrate/20260715050549_create_todos.rb`

```ruby
class CreateTodos < ActiveRecord::Migration[8.1]
  def change
    create_table :todos do |t|
      t.string  :title
      t.text    :description
      t.boolean :completed
      t.timestamps
    end
  end
end
```

Migrations are version-controlled, incremental changes to your database schema, written in Ruby instead of raw SQL. `t.timestamps` adds `created_at`/`updated_at` automatically. Running `bin/rails db:migrate` executes any migration that hasn't run yet, tracked in a `schema_migrations` table — so the whole team's databases stay in sync as the schema evolves.

To add a new field later (e.g. a due date): generate a new migration, don't edit this one.
```
bin/rails generate migration AddDueDateToTodos due_date:date
bin/rails db:migrate
```

## 5. Controller — `app/controllers/todos_controller.rb`

The controller is the traffic cop: it receives the request, talks to the model, and picks a view to render. Each public method matches one row in the routing table above.

```ruby
def index
  @todos = Todo.all
end
```
Fetches every row from `todos` and stores it in `@todos`. Instance variables (`@`-prefixed) are how controllers hand data to views — anything set in an action is automatically visible in that action's view template.

```ruby
def create
  @todo = Todo.new(todo_params)
  if @todo.save
    redirect_to @todo, notice: "Todo was successfully created."
  else
    render :new, status: :unprocessable_content
  end
end
```
Builds a `Todo` from submitted form params, tries to save it. `@todo.save` returns `true`/`false` — validations (if you add any to the model, e.g. `validates :title, presence: true`) run here automatically.

```ruby
def todo_params
  params.expect(todo: [ :title, :description, :completed ])
end
```
**Strong parameters** — this is a security control. It whitelists exactly which submitted fields are allowed to be mass-assigned to the model. Without this, a malicious form submission could set arbitrary columns (e.g. an `admin` flag if one existed).

```ruby
before_action :set_todo, only: %i[ show edit update destroy ]
```
Runs `set_todo` before those 4 actions, so `@todo = Todo.find(params[:id])` isn't repeated 4 times — a shared setup hook.

## 6. View — `app/views/todos/*.html.erb`

Views are `.erb` templates (embedded Ruby inside HTML). `<%= %>` outputs a value, `<% %>` runs Ruby without outputting anything.

`index.html.erb` loops over `@todos` (set by the controller) and renders each one:
```erb
<% @todos.each do |todo| %>
  <%= render todo %>   <!-- renders app/views/todos/_todo.html.erb for this one record -->
<% end %>
```

`_todo.html.erb` (the underscore prefix marks it a **partial** — a reusable snippet) is what actually prints one todo's fields.

`_form.html.erb` is shared between `new.html.erb` and `edit.html.erb` — one form definition used for both creating and updating, via Rails' `form_with(model: todo)` helper which automatically points at the right URL/HTTP verb depending on whether `todo` is a new or existing record.

Styling is Tailwind CSS utility classes directly in the markup (`class="rounded-md px-3.5 py-2.5 bg-blue-600 ..."`) — no separate CSS files to maintain per component.

## 7. Putting a Full Request Together

Example: submitting the "New Todo" form.

1. Browser POSTs form data to `/todos`.
2. `config/routes.rb` → `resources :todos` maps `POST /todos` to `TodosController#create`.
3. `create` runs, builds `@todo = Todo.new(todo_params)` (only whitelisted fields).
4. `@todo.save` — Active Record generates and runs an `INSERT INTO todos (...)` SQL statement against PostgreSQL.
5. If it saved, `redirect_to @todo` sends the browser to `GET /todos/:id`, which routes to `#show`, which renders `show.html.erb`.
6. If it failed (e.g. a validation), `render :new` re-renders the form with `@todo.errors` populated, so `_form.html.erb`'s error block displays what went wrong.

## 8. Where the Database Connection Comes From

`config/database.yml` doesn't hardcode a password — it reads from environment variables:
```yaml
username: <%= ENV.fetch("DATABASE_USERNAME") { "todo_app" } %>
password: <%= ENV.fetch("DATABASE_PASSWORD") { nil } %>
```
Those variables are loaded from `.env` (git-ignored) at boot by the `dotenv-rails` gem — so real credentials never get committed to git, and in production you'd set the same variable names as real environment variables/secrets instead of a `.env` file.

## 9. Directory Map (this project)

```
app/
  models/todo.rb              — data + business rules
  controllers/todos_controller.rb  — request handling
  views/todos/                — HTML templates
    index.html.erb, show.html.erb, new.html.erb, edit.html.erb
    _form.html.erb, _todo.html.erb   (partials, reused across pages)
config/
  routes.rb                   — URL → controller#action map
  database.yml                — DB connection config (reads from ENV)
db/
  migrate/                    — versioned schema changes
  seeds.rb                    — sample data for `rails db:seed`
test/
  models/, controllers/       — automated tests (Minitest, Rails default)
.env                          — local secrets (git-ignored, not committed)
```

## 10. Try It Yourself — Extend the App

Good next exercises, in order of difficulty:
1. Add a "Mark as complete" button that just toggles `completed` without opening the edit form (add a route + controller action).
2. Add a model validation: `validates :title, presence: true` in `todo.rb`, then see the error UI in `_form.html.erb` light up.
3. Add a `due_date` column via a new migration, add it to `todo_params` and the form.
4. Add sorting/filtering on `index` (e.g. show incomplete todos first) — all in `TodosController#index`, no new routes needed.
5. Run `bin/rails console` and try `Todo.create!(title: "test")`, `Todo.where(completed: false)`, `Todo.count` — this is the fastest way to learn Active Record's query methods interactively.
