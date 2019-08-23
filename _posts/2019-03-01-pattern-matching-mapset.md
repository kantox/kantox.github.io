---
layout: post
title: "Pattern Matching Empty MapSets"
description: "Long story short: Elixir MapSets are cool and shall be used widely. Empty MapSets can be pattern-matched easily."
category: hacking
tags:
  - elixir
  - tricks
  - tutorial
---

![Balcony on Passeig de Gràcia](/img/balcony.jpg)

_Elixir_ provides [`MapSet`](https://hexdocs.pm/elixir/MapSet.html) module to handle sets—lists of unique elements (shall I say “arrays” instead of lists here?). They are very handy under many circumstances. During my work on the minor update in [`Exvalibur`](https://hexdocs.pm/exvalibur/exvalibur.html), I changed the list I mistakenly used before to be `MapSet`. And there was a pitfall. Everywhere is the code I used different function clauses for empty list. But `MapSet` is a module. That said, this code

```elixir
@spec func(input :: list()) :: :ok | :error
def func([]), do: :error
def func([_ | _]), do: :ok
```

cannot be easily converted to work with `MapSet`. [`MapSet.size/1`](https://hexdocs.pm/elixir/MapSet.html#size/1) is an external function that cannot be used in guards. Of course one might appeal to `if` as a last resort, but that’d be silly. But hey, `MapSet` is _a struct_ underneath. And it keeps the values in [`map`](https://github.com/elixir-lang/elixir/blob/0558e7c92a93b9c7952464388504a437b9600875/lib/elixir/lib/map_set.ex#L45) field. So yes, we still can use different clauses and guards

```elixir
@spec func(input :: list()) :: :ok | :error
def func(%MapSet{map: map}) when map_size(map) == 0, do: :error
def func(%MapSet{}), do: :ok
```

Happy mapsetting!
