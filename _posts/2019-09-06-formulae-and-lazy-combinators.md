---
layout: post
title: "Formulæ and Lazy Combinators"
description: "Formulæ library introduction and tutorial on building lazy combinators"
category: hacking
tags:
  - elixir
  - erlang
---

![The airplace in the sky](/img/airplane-sun-sky.jpg)

### Formulæ Library

We in Fintech often require to check the values for simple arithmetic conditions,
like whether the exchange rate is greater than the desired value, or like.
These conditions are fully dynamic and we need something that we can apply
on new values coming. Imagine the client wanting to be notified when the exchange
rate for some currency pair hits `1.2:1` ratio. This would be deadly easy if
we could make it static:

```elixir
def notify?(rate) when rate > 1.2, do: true
def notify?(_), do: false
```

Once the value comes dynamically, we need a more or less robust mechanism to check
for the same condition. Using [`Code.eval_string/3`](https://hexdocs.pm/elixir/master/Code.html?#eval_string/3), while somewhat works, is compiling the condition every single
time it’s getting called. This is obviously wasting resources for no reason.

We finally came up with a [precompiled formulae](https://hexdocs.pm/formulae/Formulae.html#content). The tiny library creates the module for each input given and compiles the evaluator.

**NB** This should be used with care because module names are stored as atoms and
blindly creating modules for whatever the client wants to check might lead to
atoms DoS attack on the long run. We use a maximal allowed step of 0.01 which
gives at most 200 thousand atoms in the worst scenario.

### Lazy Combinators

But the main purpose of this writing is not the precompiled formula. For some
reason we needed to calculate permutations of the list of considerable length.
I decided to replicate ruby combinators ([`Array#combination`](https://ruby-doc.org/core/Array.html#method-i-combination) and family.) Unfortunately it was not as easy
for noticeably large numbers. Greedy static evaluation freezes on not as huge numbers.

That was expected, so I started to tackle with [`Stream`](https://hexdocs.pm/elixir/master/Stream.html). That was not as straightforward as I thought. I came up with something like the code below

```elixir
list = ~w[a b c d e]a

combinations =
  Stream.transform(Stream.with_index(list), :ok, fn {i1, idx1}, :ok ->
    {Stream.transform(Stream.with_index(list), :ok, fn
        {_, idx2}, :ok when idx2 <= idx1 ->
          {[], :ok}

        {i2, idx2}, :ok ->
          {Stream.transform(Stream.with_index(list), :ok, fn
            {_, idx3}, :ok when idx3 <= idx2 ->
              {[], :ok}

            {i3, idx3}, :ok ->
              {Stream.transform(Stream.with_index(list), :ok, fn
                  {_, idx4}, :ok when idx4 <= idx3 ->
                    {[], :ok}

                  {i4, _idx4}, :ok ->
                    {[[i1, i2, i3, i4]], :ok}
                end), :ok}
          end), :ok}
      end), :ok}
  end)
```

It works, but it is hardcoded for number of combinations! So, that is what
we have macros for, isn’t it?

The code has three different patterns. The successful path, emitting the list.
The emit-empty-fast clauses. And the `Stream.transform(Stream.with_index(list) ...`
clauses. Looks like we might try to generate the AST for the above.

This is the rare case when using [`Kernel.SpecialForms.quote/2`](https://hexdocs.pm/elixir/master/Kernel.SpecialForms.html?#quote/2) would probably make things more complicated, so
I went for plain old good bare AST generation. I started with quoting the above
and examining the result. Yes, there are patterns.

So, let’s start with producing the scaffold.

```elixir
defmacrop mapper(from, to, fun),
  do: quote(do: Enum.map(Range.new(unquote(from), unquote(to)), unquote(fun)))

@spec combinations(list :: list(), count :: non_neg_integer()) :: {Stream.t(), :ok}
defmacro combinations(l, n) do
  Enum.reduce(n..1, {[mapper(1, n, &var/1)], :ok}, fn i, body ->
    stream_combination_transform_clause(i, l, body)
  end)
end
```

Now we need to dive into AST to extract patterns. That was fun!

Helpers to simplify the code after.

```elixir
def var(i), do: {:"i_#{i}", [], Elixir}
def idx(i), do: {:"idx_#{i}", [], Elixir}
```

Inner clause AST.

```elixir
def sink_combination_clause(i) when i > 1 do
  {:->, [],
    [
      [
        {:when, [],
        [
          {​{:_, [], Elixir}, idx(i)},
          :ok,
          {:<=, [context: Elixir, import: Kernel], [idx(i), idx(i - 1)]}
        ]}
      ],
      {[], :ok}
    ]}
end
```

All inner clauses together.

```elixir
def sink_combination_clauses(1, body) do
  [{:->, [], [[{var(1), idx(1)}, :ok], body]}]
end

def sink_combination_clauses(i, body) when i > 1 do
  Enum.reverse([
    {:->, [], [[{var(i), idx(i)}, :ok], body]}
    | Enum.map(2..i, &sink_combination_clause/1)
  ])
end
```

And, finally, the outer clause.

```elixir
def stream_combination_transform_clause(i, l, body) do
  clauses = sink_combination_clauses(i, body)

  {​{​{:., [], [{:__aliases__, [alias: false], [:Stream]}, :transform]}, [],
    [
      {​{:., [], [{:__aliases__, [alias: false], [:Stream]}, :with_index]}, [], [l]},
      :ok,
      {:fn, [], clauses}
    ]}, :ok}
end
```

Permutations are done almost the same, the only change is the condition
in the inner clauses. That was easy!

### Application

OK, so what can we do with this in place? Somewhat like this.

```elixir
l = for c <- ?a..?z, do: <<c>> # letters list
with {stream, :ok} <- Formulae.Combinators.Stream.permutations(l, 12),
  do: stream |> Stream.take_every(26) |> Enum.take(2)

#⇒ [["a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l"],
#   ["a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "l", "w"]]
```

We now even can feed a [`Flow`](https://hexdocs.pm/flow/Flow.html) with a stream
returned above and spawn a process that will unhurriedly walk through all the
combinations doing something business important.

Happy lazy permutating!
