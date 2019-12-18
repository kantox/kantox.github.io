---
layout: post
title: "ActiveRecord Smell With Elixir/Ecto"
description: "Quick how-to on bringing some ActiveRecord goodness into Elixir/Ecto"
category: hacking
tags:
  - ruby
  - tricks
  - elixir
---

## What? Why?

[`ActiveRecord`](https://api.rubyonrails.org/classes/ActiveRecord.html) is arguably the worst approach to deal with the database ever existed. While object-relational mapping definitely might bring some goodness to our lives, the rules _AR_ is built upon can sometimes be shooting our legs.

[`Ecto`](https://hexdocs.pm/ecto/3.0.0-rc.1) was really refreshing for everybody suffering from the necessity to conform wild guesses constantly made by _AR_. Unlike the latter, `Ecto` is deadly explicit. Nothing happens under the hood. It performs exactly what the developer wanted to be performed. No gazillions of unsolicited queries, no hidden DB updates when serialized fields change. It’s amazing.

After the year of a pure rapture, I found myself in a need to duplicate some boilerplate amongst different projects. So I decided to implement the most vivid (read: the best) parts of what _AR_ provides in _Elixir_ for _Ecto_.

Maybe some day I’ll build a package out of this code, now I want to share the way I had it implemented. This would be a tutorial on how to write macros in _Elixir_ even more than reimplementation of _AR_ helpers for the sake of decreasing the boilerplate.

Yes, some things from _AR_ are good enough for me to miss them.

## Goal

What we want is to have something like this

```elixir
defmodule Post do
  use EctoAR.Query,
    repo: MyApp.Repo,
    preload: [:user, :comments],
    enums: [state: ~w|draft published|]
...
```

to have some AR-like goodness for free. What I am after in this writing is:

- `Post.find(id)` → `%Post{}`
- `Post.scope(` [`Ecto.Queryable`](https://github.com/elixir-ecto/ecto/blob/v3.0.0-rc.1/lib/ecto/queryable.ex) `, params)` → [`%Ecto.Query{}`](https://hexdocs.pm/ecto/3.0.0-rc.1/Ecto.Query.html#content)
- `Post.scope!(Ecto.Queryable, params)` → `[%Post{}]`

and some helpers for fun:

- `Post.state_draft(params)` → `[%Post{}]`
- `Post.state_published(params)` → `[%Post{}]`

## Warning Note

If you are not familiar with Elixir Macros, I’d strongly suggest reading the brilliant [Metaprogramming Elixir](https://pragprog.com/book/cmelixir/metaprogramming-elixir) by Chris McCord. It is absolutely worth it even if you are not elixiring on daily basis.

## Implementation

### Scaffold

So, we are to produce a macro that will inject the boilerplate into our bare _Ecto_ schemas. Since I want to use it as `use EctoAR.Query`, that’ll be a module named `EctoAR.Query`, implementing `__using__/1` macro. Let’s start with a scaffold.

```elixir
defmodule EctoAR.Query do
  @moduledoc false
  defmacrop repo(opts) do
    # try `opts`, then try the topmost application’s `config`, then fallback `Repo`
    quote bind_quoted: [opts: opts], location: :keep do
      with nil <- Keyword.get(opts, :repo),
           [{me, _, _} | _] = Application.started_applications(),
           nil <- Application.get_env(me, :repo),
           do: Module.concat(Macro.camelize(to_string(me)), "Repo")
    end
  end

  defmacrop ast() do
    quote do
      ???
    end
  end

  defmacro __using__(opts \\ []) do
    repo = repo(opts)
    preload =
      opts
      |> Keyword.get(:preload, [])
      |> Macro.expand(__CALLER__)

    [
      quote do
        import(Ecto.Query)
        @type t :: %__MODULE__{}
      end
      | ast()
    ]
  end
end
```

The above is basically a scaffold we are to fill with the desired functionality. First comes the private macro to do our best in resolving the repository we are to use to connect to our data. It first tries to get the `:repo` keyword parameter from options passed to `use EctoAR.Query` call, then looks up the application config and finally falls back to default `MyApp.Repo`.

During the compilation stage, we cannot ensure the value provided is legit. It is possible to raise a meaningful exception with a solid description of what’s wrong (and I actually have it in the resulting code,) but for the sake of example let’s assume we are fine with reading docs and providing correct arguments in the call to our macro. For those curious how to implement graceful error in runtime when repo specified cannot be found, check [`Code.ensure_loaded/1`](https://hexdocs.pm/elixir/master/Code.html#ensure_loaded/1).

Also, I’d explicitly note the wise use of [`Kernel.SpecialForms.with/1`](https://hexdocs.pm/elixir/master/Kernel.SpecialForms.html#with/1) _monadic_ clause. It immediately returns a value as it was found and continues to the next clause otherwise.

### `scope/2`

The idea of `scope/2` is to behave nearly the same as AR’s `where` does, but with an explicit check for arguments passed. It should accept any queryable and return a queryable back to provide full support for embedding it in query chains. Here we go.

```elixir
defmacrop scopes(preload) do
  quote do
    @spec scope(
      query :: nil | Ecto.Query.t(),
      params :: Keyword.t()
    ) :: Ecto.Query.t()

    @doc """
    Returns an `Ecto.Query` for `where(params)`.
    If the first parameter in call to this method
      is omitted, defaults to `#{__MODULE__}`.
    """

    def scope(query \\ nil, params)

    @spec do_scope(
      query :: nil | Ecto.Query.t(),
      params :: Keyword.t()
    ) :: Ecto.Query.t()

    defp do_scope(query, params) do
      case Keyword.keys(params) -- __schema__(:fields) do
        [] ->
          from(data in query, where: ^params)
          |> preload(unquote(preload))

        extra ->
          raise """
            Unsupported fields were passed in call to `#{__MODULE__}__.scope/2`.
            Extra fields: #{inspect(extra)}
          """
      end
    end

    def scope(nil, params), do: do_scope(__MODULE__, params)
    def scope(%Ecto.Query{} = query, params), do: do_scope(query, params)

    defoverridable(scope: 2)
  end
end
```

The very last method should actually accept any implementation of [`Ecto.Queryable`](https://github.com/elixir-ecto/ecto/blob/v3.0.0-rc.1/lib/ecto/queryable.ex#L1), but for the sake of the example, let it be just accepting queries.

OK, once the AST above is injected into `Ecto.Schema` instance, the latter receives the ability to call this function like:

```elixir
Post.scope(author_id: 10, date: ~D[2018-10-30])
#⇒ #Ecto.Query<from data in Post,
#   where: data.author_id == ^10 and data.date == ^~D[2018-10-30],
#   preload: [:user, :comments]>
```

Note, that an attempt to pass the filter for inexisting field leads to the exception raised:

```elixir
Post.scope(author_id: 10, answer: 42)
#⇒ ** (RuntimeError) Unsupported fields were passed in call to `Elixir.Post.scope/2`.
#   Extra fields:
#     [answer: 42]
```

Also note, that I made `scope/2` overridable FWIW.

### `find/1`

That would be easy. In the real life, it is more complicated, since the primary key might be either inexistent or combined, and we would need to tackle with `__schema__(:primary_key)` and [`Kernel.SpecialForms.unquote_splicing/1`](https://hexdocs.pm/elixir/master/Kernel.SpecialForms.html#unquote_splicing/1), but for the sake of this example let’s keep it simple.

```elixir
defmacrop find(repo, preload) do
  quote do
    def find(id) when is_integer(id) do
      from(data in __MODULE__, where: data.id == ^id)
      |> preload(unquote(preload))
      |> unquote(repo).one()
    end
  end
end
```

Use it as:

```elixir
Post.find(100)
#⇒ %Post{
#   __meta__: #Ecto.Schema.Metadata<:loaded, "posts">,
#   user: %User{
#     ...
#   },
# ...
```

### Anything Else?

One might actually implement and inject as many helper methods as needed, but we are to stop here. I also have `scope!/2` that wraps the call to `scope/2` with actuall request against the database, returning _records_ not a query.

```elixir
@spec scope!(
  query :: nil | Ecto.Query.t(), params :: Keyword.t()
) :: [Ecto.Schema.t()]

@doc ~s|Returns a dataset for `where(params)`|

def scope!(query \\ nil, params),
  do: apply(unquote(repo), :all, [scope(query, params)])
```

### Helpers For Enums

This is completely redundant, but let’s make them to show the power of _Elixir_ macros again.

```elixir
defmacrop enums(opts) do
  opts
  |> Keyword.get(:enums, [])
  |> Enum.flat_map(fn {prefix, values} ->
    values
    |> Macro.expand(__CALLER__) # to allow sigils
    |> Enum.map(fn value ->
      quote do
        @spec unquote(:"#{prefix}_#{value}")(opts :: Keyword.t()) :: [Ecto.Schema.t()]
        @doc ~s|Returns a dataset for `where(#{unquote(prefix)}: "#{unquote(value)}")`|
        defp unquote(:"#{prefix}_#{value}")(opts) do
          (data in scope(opts))
          |> from(where: data.unquote(prefix) == ^unquote(value))
          |> unquote(repo).all()
        end
      end
    end)
  end)
end
```

Note, that specs and docs will be _caller specific_, including real names of functions for these particular enums.

### Putting It All Together

All we need to do to make it work, would be to build the AST in the scaffold mentioned above to be injected into our schemas. That would be as easy as

```elixir
ast =
  [
    quote do
      import(Ecto.Query)
      @type t :: %__MODULE__{}
    end
    | [find(repo, preload) | scopes(preload) ++ enums(opts)]
  ]
```

We are ready to `use` our helper in _Ecto_ schemas. Happy macroing!
