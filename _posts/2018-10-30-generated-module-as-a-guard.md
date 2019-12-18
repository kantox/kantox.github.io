---
layout: post
title: "Generated Module As A Guard"
description: "Better and faster way to implement validation of incoming data than just looking it up in the global config map"
category: hacking
tags:
  - elixir
  - tricks
  - tools
---

Imagine the application that receives some data from the external source. For the sake of an example let’s assume the data is currency rates stream.

The application has a set of rules to filter the incoming data stream. Let’s say we have a list of currencies we are interested in, and we want only the currencies from this list to pass through. Also, sometimes we receive invalid rates (nobody is perfect, our rates provider is not an exception.) So we maintain a long-lived validator that ensures that the rate in the stream looks fine for us and only then we allow the machinery to process it. Otherwise we just ignore it.

## Naïve Approach

The naïve approach to handle this use case would be to maintain a map, containing currency pairs as keys and rules as values, and apply rules to the incomind rates to check whether the rate is of interest or not. Rules would be simple maps specifying the acceptable interval for the rate as `min` and `max` values. Something like this (for the sake of an example let’s assume rates are coming as maps already, for instance from RabbitMQ or like):

```elixir
defmodule Validator do
  use Agent

  def start_link,
    do: Agent.start_link(fn -> %{} end, name: __MODULE__)

  def update_rules(currency_pair, rules),
    do: Agent.update(__MODULE__, &(Map.put(&1, currency_pair, rules))

  def valid?(%{currency_pair: pair, rate: rate}) do
    with %{} = rules <- Agent.get(__MODULE__, & &1),
         rule when not is_nil(rule) <- rules[pair],
         r when r > rule.min and r < rule.max  <- rate do
      {:ok, r},
    else
      _ -> :error
    end
  end
end
```

This is good, and this works. But can we improve the performance?—Sure thing!

## Pattern matching

Instead of looking up the map with rules, we might simply generate the module, that will have one function `valid?` with as many clauses as we have rules. These clauses will directly pattern match the input and return `{:ok, rate}` tuple (the _existence_ of this clause guarantees that the rate is good.) The last clause will accept all the non-matched garbage to return `:error`.

Sounds smart?—Indeed. Let’s implement it. Imagine we still have this map with rules.

```elixir
defmodule Validator do
  def instance!(rules) do
    mod = Module.concat(["Validator", "Instance"])

    if Code.ensure_compiled?(mod) do
      :code.purge(mod)
      :code.delete(mod)
    end

    Module.create(mod, ast(rules), Macro.Env.location(__ENV__))
  end

  defp ast(map) do
    [
      quote(do: (def valid?(_), do: :error)) |
      Enum.map(map, fn {pair, %{min: min, max: max}} ->
        quote do
          def valid?(%{currency_pair: unquote(pair), rate: rate})
                when rate > unquote(min) and rate < unquote(max),
            do: {:ok, rate}
        end
      end)
    ] |> :lists.reverse()
  end
end
```

When we need to update our rules, we just call `Validator.instance!/1` and receive back the module named `Validator.Instance`. It has several clauses for `valid/1` function. Let’s see this in action

```elixir
iex|1 ▶ Validator.instance!(%{"USDEUR" => %{min: 1.0, max: 2.0}})
{:module, Validator.Instance,
 <<70, 79, 82, 49, 0, 0, 4, 244, 66, 69, 65, 77, 65, 116, 85, 56, 0, 0, 0, 167,
   0, 0, 0, 17, 25, 69, 108, 105, 120, 105, 114, 46, 86, 97, 108, 105, 100, 97,
   116, 111, 114, 46, 73, 110, 115, 116, 97, ...>>, [valid?: 1, valid?: 1]}

iex|2 ▶ Validator.Instance.valid?(%{currency_pair: "USDEUR", rate: 1.5})
{:ok, 1.5}

iex|3 ▶ Validator.Instance.valid?(%{currency_pair: "USDEUR", rate: 0.5})
:error

iex|4 ▶ Validator.Instance.valid?(%{currency_pair: "USDGBP", rate: 1.5})
:error
```

Exactly what we needed, and blazingly fast.

## Further Improvement

If the amount of incoming rates is big enough, like thousands per a second, we might use [`Flow`](https://hexdocs.pm/flow) on [`GenStage`](https://hexdocs.pm/gen_stage) to validate them in bulks. That would be probably the topic of the next writing on the subject.

Happy generating!
