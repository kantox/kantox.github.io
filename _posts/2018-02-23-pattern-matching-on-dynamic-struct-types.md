---
layout: post
title: "Pattern matching on dynamic struct types"
description: "A less known patter matching feature in Elixir"
category: hacking
tags:
  - elixir
---

Pattern matching is great.

Strictly speaking, I could end this post right here, but occasionally
I have an interesting Elixir feature on hand. That is related to pattern
matching. That is, I bet, not widely known at all.

One can pattern match on dynamic struct type with pin operator
[`Kernel.SpecialForms.^/1`](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#%5E/1).
It’s documentation says:

> Accesses an already bound variable in match clauses. Also known as the pin operator.

Not quite expressive, neither informative. But while a documentation walks, the
code talks. Check this:

```elixir
iex|1 ▶ defmodule MyMod, do: defstruct ~w|foo bar|a
iex|2 ▶ mod_ok = MyMod
iex|3 ▶ mod_ko = Integer
iex|4 ▶ %^mod_ok{} = %MyMod{foo: 42, bar: 3.14}
#⇒ %MyMod{bar: 3.14, foo: 42}
iex|5 ▶ %^mod_ko{} = %MyMod{foo: 42, bar: 3.14}
#⇒ ** (MatchError) no match of right hand side value:
#       %MyMod{bar: 3.14, foo: 42}
```

Wow. We can explicitly pattern match on dynamic struct types! It also works
in `case` clauses:

```elixir
iex|6 ▶ case %MyMod{foo: 42, bar: 3.14} do
...|6 ▶   %^mod_ok{} = %{foo: _foo} ->
...|6 ▶     IO.inspect(mod_ok, label: "Pinned module")
...|6 ▶ end
#⇒ Pinned module: MyMod
```

FWIW, the latter might be used without a _pin oerator_ to get the struct type

```elixir
iex|7 ▶ case %MyMod{foo: 42, bar: 3.14} do
...|7 ▶   %mod{} -> IO.inspect(mod, label: "Matched module")
...|7 ▶ end
#⇒ Matched module: MyMod
```

This match is the cumbersome spelling of `%MyMod{foo: 42, bar: 3.14}.__struct__`,
though.

---

Permalink to the Elixir codebase:
[MapTest.exs](https://github.com/elixir-lang/elixir/blob/cbde356d104996e082b1752b559a64e5c6576f51/lib/elixir/test/elixir/map_test.exs#L205).
