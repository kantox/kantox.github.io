---
layout: post
title: "Elixir Pipeline Operators"
description: "What operators can be used in Elixir for pipelining and how"
category: hacking
tags:
  - elixir
  - tricks
---

Yesterday I was answering [this SO question](https://stackoverflow.com/a/49314637/2035262),
roughly asking for how to use an uncommon syntax in pipe operator.

“Man, that’s Elixir,” was my very first response. “We have macros for that.”
The pitfall here is: while we can restrict exporting the standard
[`Kernel.|>/2`](https://hexdocs.pm/elixir/Kernel.html#%7C%3E/2) and overload it,
this approach does not sound as a good proposal since we’ll lose all the nifty
default functionality. Of course, one might check the right operand and if it’s
one of those expected by standard pipe, fallback to default `Kernel.|>(right)`,
but this is not nifty at all.

So, I grepped Elixir source code for _“pipeline”_ keyword and [found this](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/lib/code/formatter.ex#L22):

```elixir
@pipeline_operators [:|>, :~>>, :<<~, :~>, :<~, :<~>, :<|>]
```

I was unable to _duckduckgo_ anything related to what are permitted pipe
operators in Elixir, and I even am not sure this behaviour being an
implementation detail or is it guaranteed to remain the same, but I wrote
some checks and all those are working pretty fine as pipe operators in all
the modern versions of Elixir I have on hand.

I am going to show how to use the custom pipe operator in your codebase
to make the code looking sexier, in a case you have a colleague to impress.
Let’s stay with the question as it was stated on SO:

> There is a need to update the struct without breaking the pipeline up.
> Something like this:
>
>     my_struct
>     |> %{ | my_field_in_struct: a_new_value}
>     |> my_funct1
>     |> %{ | my_field_in_struct: a_new_value}
>     |> my_funct2
>     |> %{ | my_field_in_struct: a_new_value}
>     |> my_funct3

Let’s start with introducing our own pipe operator implementation:

```elixir
defmodule StructPiper do
  defmacro __using__(_) do
    quote do
      require unquote(__MODULE__)
      import unquote(__MODULE__)
    end
  end

  # TODO: implement this
  defmacro left ~>> right do
    IO.inspect {left, right}, label: "DEBUG"
    :ok
  end
end
```

The code above might be used like this:

```elixir
defmodule MyStruct do
  defstruct ~w|foo bar baz|a
end

defmodule StructPiper.Test do
  use StructPiper
  def test do
    %MyStruct{foo: 42}
    |> IO.inspect(label: "1")
    ~>> [bar: 3.14]
    |> IO.inspect(label: "2")
    ~>> [baz: "FOOBAR"]
  end
end

IO.inspect(StructPiper.Test.test ==
  %MyStruct{bar: 3.14, baz: "FOOBAR", foo: 42}, label: "Test")
```

Let’s run the test to check what do we have there:

```elixir
iex|1 ▶ m = %MyStruct{foo: 42}
iex|2 ▶ m ~>> [bar: 3.14]
#⇒ DEBUG: {​{:m, [line: 24], nil}, [bar: 3.14]}
```

OK, we receive `left` and `right` operands as expected. The only thing we need
would be to transform them to the AST for `%{left | right}`. Let’s check
how the latter looks like:

```elixir
iex|3 ▶ quote do: %{m | bar: 3.14}
{:%{}, [], [{:|, [], [{:m, [], Elixir}, [bar: 3.14]]}]}
```

Check the last part of the AST related to `:|` operator in this map/struct
updating syntax: arguments are _exactly_ the same as we got in the call to
our pipe macro. That said, in this particular case it would be easier not to
tackle with quoting/unquoting and return the AST out of the box:

```elixir
defmacro left ~>> right do
  {:%{}, [], [{:|, [], [left, right]}]}
end
```

We’re all set! Below is the full working implementation of the pipe operator
macro `:~>>`:

```elixir
defmodule StructPiper do
  defmacro __using__(_) do
    quote do
      require unquote(__MODULE__)
      import unquote(__MODULE__)
    end
  end

  defmacro left ~>> right do
    {:%{}, [], [{:|, [], [left, right]}]}
  end
end
```

If you don’t like this particular pipe operator, feel free to pick another
from the list this post starts with.
