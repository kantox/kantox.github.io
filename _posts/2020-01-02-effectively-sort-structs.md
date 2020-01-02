---
layout: post
title: "Effectively Sort Structs"
description: "Structs can now be sorted with Enum.sort/2"
category: hacking
tags:
  - elixir
  - tricks
---

One coming to _Elixir_/_Erlang_ from other languages might have some expectations on how comparison operators `<`, `>`, `==` etc. should work. One might expect `1 < 2` to be `true` (which is indeed true). Well, actually comparison works as we expect in _Elixir_/_Erlang_. Unless it does not.

We can compare _anything_. While for two operands of the same type the result is not quite surprising, as in the example above, for operands of different types we have [term ordering](https://hexdocs.pm/elixir/master/operators.html#term-ordering) as shown below

```elixir
number < atom < reference < function < port < pid < tuple < map < list < bitstring
```

Which effectively makes `42 < nil #⇒ true`.

Also, maps (and therefore structs, which are bare maps underneath,) are compared according to their fields in alphabetical order. That said, the dates (which are represeted by [`Date`](https://hexdocs.pm/elixir/master/Date.html) struct in _Elixir_) will be compared by `day`, `month` and then `year`, which is the opposite of what we probably wanted.

---

Starting with v1.10.0, Elixir provides handy sorting with [Enum.sort/2](https://hexdocs.pm/elixir/master/Enum.html#sort/2-sorting-structs) for structs implementing `compare/2`:

```elixir
defmodule User do
  defstruct [:name]
  def compare(%User{name: n1}, %User{name: n2}) when n1 < n2,
    do: :lt
  def compare(%User{name: n1}, %User{name: n2}) when n1 > n2,
    do: :gt
  def compare(%User{}, %User{}), do: :eq
end

users = [
  %User{name: "john"},
  %User{name: "joe"},
  %User{name: "jane"}
]

Enum.sort(users, {:asc, User})
#⇒ [%User{name: "jane"},
#   %User{name: "joe"},
#   %User{name: "john"}]
```

Any module that both _defines struct_, and _exports `compare/2` function_ might be passed as the second parameter in call to `Enum.sort/2` either as is, or as `{:asc | :desc, StructModule}` tuple. `Enum.sort/2` is now smart enough to [call `compare/2` of the module passed](https://github.com/elixir-lang/elixir/blob/ee758f987b7e240754bf386b903b92d38fb02233/lib/elixir/lib/enum.ex#L2515-L2517) as a sorter function. Below is the excerpt from `Enum` module

```elixir
...
defp to_sort_fun(module) when is_atom(module), do: &(module.compare(&1, &2) != :gt)
defp to_sort_fun({:asc, module}) when is_atom(module), do: &(module.compare(&1, &2) != :gt)
defp to_sort_fun({:desc, module}) when is_atom(module), do: &(module.compare(&1, &2) != :lt)
```

That makes it possible to properly sort `Date`s in the example below (because `Date` exports `compare/2` function) as well as any custom structure (given it exports `compare/2` function,) as in the example with `User` struct above.

```elixir
dates = [~D[2019-01-01], ~D[2020-03-02], ~D[2019-06-06]]

Enum.sort(dates) # wrong
#⇒ [~D[2019-01-01], ~D[2020-03-02], ~D[2019-06-06]]

Enum.sort(dates, {:asc, Date}) # correct
#⇒ [~D[2019-01-01], ~D[2019-06-06], ~D[2020-03-02]]
```

Happy comparing!
