Get Started With Postgres

## Goals

In this guide we will:

1. Setup AshPostgres, which includes setting up [Ecto](https://hexdocs.pm/ecto/Ecto.html)
2. Add AshPostgres to the resources created in {{link:ash:guide:Get Started}}
3. Show how the various features of AshPostgres can help you work quickly and cleanly against a postgres database
4. Highlight some of the more advanced features you can use when using AshPostgres.
5. Point you to additional resources you may need on your journey

## Things you may want to read

- [Install PostgreSQL](https://www.postgresql.org/download/) (I recommend the homebrew option for mac users)

## Requirements

- A working Postgres installation, with a sufficiently permissive user
- If you would like to follow along, you will need to add begin with {{link:ash:guide:Get Started}}

## Steps

### Add AshPostgres

Add the `:ash_postgres` dependency to your application

{{mix_dep:ash_postgres}}

Add `:ash_postgres` to your `.formatter.exs` file

```elixir
[
  # import the formatter rules from `:ash_postgres`
  import_deps: [..., :ash_postgres],
  inputs: [...]
]
```

### Create and configure your Repo

Create `lib/helpdesk/repo.ex` with the following contents. `AshPostgres.Repo` is a thin wrapper around `Ecto.Repo`, so see their documentation for how to use it if you need to use it directly. For standard Ash usage, all you will need to do is configure your resources to use your repo.

```elixir
# in lib/helpdesk/repo.ex

defmodule Helpdesk.Repo do
  use AshPostgres.Repo, otp_app: :helpdesk
end
```

Next we will need to create configuration files for various environments. Run the following to create the configuration files we need.

```bash
mkdir -p config
touch config/config.exs
touch config/dev.exs
touch config/runtime.exs
touch config/test.exs
```

Place the following contents in those files, ensuring that the credentials match the user you created for your database. For most conventional installations this will work out of the box. If you've followed other guides before this one, they may have had you create these files already, so just make sure these contents are there.

```elixir
# in config/config.exs
import Config

config :helpdesk,
  ash_apis: [Helpdesk.Tickets]

config :helpdesk,
  ecto_repos: [Helpdesk.Repo]

# Import environment specific config. This must remain at the bottom
# of this file so it overrides the configuration defined above.
import_config "#{config_env()}.exs"
```

```elixir
# in config/dev.exs

import Config

# Configure your database
config :helpdesk, Helpdesk.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  database: "helpdesk_dev",
  port: 5432,
  show_sensitive_data_on_connection_error: true,
  pool_size: 10
```

```elixir
# in config/runtime.exs

import Config

if config_env() == :prod do
  database_url =
    System.get_env("DATABASE_URL") ||
      raise """
      environment variable DATABASE_URL is missing.
      For example: ecto://USER:PASS@HOST/DATABASE
      """

  config :helpdesk, Helpdesk.Repo,
    url: database_url,
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10")
end
```

```elixir
# in config/test.exs

import Config

# Configure your database
#
# The MIX_TEST_PARTITION environment variable can be used
# to provide built-in test partitioning in CI environment.
# Run `mix help test` for more information.
config :helpdesk, Helpdesk.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  database: "helpdesk_test#{System.get_env("MIX_TEST_PARTITION")}",
  pool: Ecto.Adapters.SQL.Sandbox,
  pool_size: 10
```

And finally, add the repo to your application

```elixir
  def start(_type, _args) do
    children = [
      # Starts a worker by calling: Helpdesk.Worker.start_link(arg)
      # {Helpdesk.Worker, arg}
      Helpdesk.Repo
    ]

    ...
```

### Add AshPostgres to our resources

Now we can add the data layer to our resources. The basic configuration for a resource requires the {{link:ash_postgres:option:ashpostgres/postgres/table}} and the {{link:ash_postgres:dsl:ash_postgres/postgres/repo}}.

```elixir
# in lib/helpdesk/tickets/resources/ticket.ex

  use Ash.Resource,
    data_layer: AshPostgres.DataLayer

  postgres do
    table "tickets"
    repo Helpdesk.Repo
  end
```

```elixir
# in lib/helpdesk/tickets/resources/representative.ex

  use Ash.Resource,
    data_layer: AshPostgres.DataLayer

  postgres do
    table "representatives"
    repo Helpdesk.Repo
  end
```

### Create the database and tables

First, we'll create the database with `mix ash_postgres.create`.

Then we will generate database migrations. This is one of the many ways that AshPostgres can save time and reduce complexity.

```bash
mix ash_postgres.generate_migrations --name add_tickets_and_representatives
```

If you are unfamiliar with database migrations, it is a good idea to get a rough idea of what they are and how they work. See the links at the bottom of this guide for more. A rough overview of how migrations work is that each time you need to make changes to your database, they are saved as small, reproducible scripts that can be applied in order. This is necessary both for clean deploys as well as working with multiple developers making changes to the structure of a single database.

Typically, you need to write these by hand. AshPostgres, however, will store snapshots each time you run the command to generate migrations and will figure out what migrations need to be created.

You should always look at the generated migrations to ensure that they look correct. Do so now by looking at the generated file in `priv/repo/migrations`.

Finally, we will apply the generated migrations to our local database:

```bash
mix ash_postgres.migrate
```

### Try it out

And now we're ready to try it out! Run the following in iex:

Lets create some data. We'll make a representative and give them some open and some closed tickets.

```elixir
require Ash.Query

representative = (
  Helpdesk.Tickets.Representative
  |> Ash.Changeset.for_create(:create, %{name: "Joe Armstrong"})
  |> Helpdesk.Tickets.create!()
)

for i <- 0..5 do
  ticket = 
    Helpdesk.Tickets.Ticket
    |> Ash.Changeset.for_create(:open, %{subject: "Issue #{i}"})
    |> Helpdesk.Tickets.create!()
    |> Ash.Changeset.for_update(:assign, %{representative_id: representative.id})
    |> Helpdesk.Tickets.update!()
  
  if rem(i, 2) == 0 do
    ticket
    |> Ash.Changeset.for_update(:close)
    |> Helpdesk.Tickets.update!()
  end
end
```

And now we can read that data. You should see some debug logs that show the sql queries AshPostgres is generating.

```elixir
require Ash.Query

# Show the tickets where the subject contains "2"
Helpdesk.Tickets.Ticket
|> Ash.Query.filter(contains(subject, "2"))
|> Helpdesk.Tickets.read!()
```

```elixir
require Ash.Query

# Show the tickets that are closed and their subject does not contain "4"
Helpdesk.Tickets.Ticket
|> Ash.Query.filter(status == :closed and not(contains(subject, "4")))
|> Helpdesk.Tickets.read!()
```

And, naturally, now that we are storing this in postgres, this database is persisted even if we stop/start our application. The nice thing, however, is that this was the *exact* same code that we ran against our resources when they were backed by ETS.

### Aggregates

Lets add some aggregates to our representatives resource. Aggregates are a tool to include grouped up data about relationships. You can read more about them in the {{link:ash:guide:Aggregates}} guide.

```elixir
# in lib/helpdesk/tickets/resources/representative.ex

  aggregates do
    # The first argument here is the name of the aggregate
    # The second is the relationship
    count :total_tickets, :tickets

    count :open_tickets, :tickets do
      # Here we add a filter over the data that we are aggregating
      filter expr(status == :open)
    end

    count :closed_tickets, :tickets do
      filter expr(status == :closed)
    end
  end
```

Aggregates are powerful because they will be translated to SQL, and can be used in filters and sorts. For example:

```elixir
require Ash.Query

Helpdesk.Tickets.Representative
|> Ash.Query.filter(closed_tickets < 4)
|> Ash.Query.sort(closed_tickets: :desc)
|> Helpdesk.Tickets.read!()
```

You can also load individual aggregates on demand after queries have already been run, and minimal SQL will be issued to run the aggregate.

```elixir
require Ash.Query

Helpdesk.Tickets.Representative
|> Ash.Query.filter(closed_tickets < 4)
|> Ash.Query.sort(closed_tickets: :desc)
|> Helpdesk.Tickets.read!()
```

```elixir
tickets = Helpdesk.Tickets.read!(Helpdesk.Tickets.Representative)

Helpdesk.Tickets.load!(tickets, :open_tickets)
```

### Calculations

Calculations can be pushed down into SQL in the same way. Calculations are similar to aggregates, except they work on individual records. They can, however, refer to calculations on the resource, which opens up powerful possibilities with very simple code.

For example, we can determine the percentage of tickets that are open:

```elixir
# in lib/helpdesk/tickets/resources/representative.ex

  calculations do
    calculate :percent_open, :float, expr(open_tickets / total_tickets ) 
  end
```

Calculations can be loaded and used in the same way as aggregates.

```elixir
require Ash.Query

Helpdesk.Tickets.Representative
|> Ash.Query.filter(percent_open > 0.25)
|> Ash.Query.sort(:percent_open)
|> Ash.Query.load(:percent_open)
|> Helpdesk.Tickets.read!()
```

### Rich Configuration Options

Take a look at the DSL documentation for more information on what you can configure. You can add check constraints, configure the behavior of foreign keys, use postgres schemas with Ash's {{link:ash:guide:Multitenancy}} feature, and more!

### What next?

#### See what is available in the DSL

{{link:ash_postgres:dsl:ashpostgres:postgres}}

#### Learn more about the underlying tools

- [Ecto's documentation](https://hexdocs.pm/ecto/Ecto.html). AshPostgres (and much of Ash itself) is made possible by the amazing Ecto. If you find yourself looking for escape hatches when using Ash or ways to work directly with your database, you will want to know how Ecto works. Ash and AshPostgres intentionally do not hide Ecto, and in fact encourages its use whenever you need an escape hatch.

- [Postgres' documentation](https://www.postgresql.org/docs/). Although AshPostgres makes things a lot easier, you generally can't get away with not understanding the basics of postgres and SQL.

- [Ecto's Migration documentation](https://hexdocs.pm/ecto_sql/Ecto.Migration.html) read more about migrations. Even with the ash_postgres migration generator, you will very likely need to modify your own migrations some day.
