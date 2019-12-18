---
layout: post
title: "Dialyzer specs: 2 in 1"
description: "Arguably more elegane way to save keystrokes when providing specs"
category: hacking
tags:
  - elixir
  - erlang
---

![Montjuïc](/img/montjuik.jpg)

There are two types of erlang and elixir developers: those who write specs for _Dialyzer_ and those who don't _yet_. At first glance, it seems to be a waste of time, especially for those who come from languages with lax typing. However, specs have helped me catch several bugs even before the CI stage, and-sooner or later-any developer realizes that they are indeed a goodness. Not only as a tool to guide semi-strict typing, but also as a great help in documenting the code.

But, as is always happens in our cruel world, there are drawbacks. Essentially, the `@spec` directives duplicate the function declaration code. Below I am going to demonstrate how twenty lines of code can help to combine a specification and a function declaration into a nifty

```elixir
defs is_forty_two(n: integer) :: boolean do
  n == 42
end
```


They say, there is nothing in _Elixir_, but macros. Even [`Kernel.defmacro/2`](https://hexdocs.pm/elixir/master/Kernel.html?#defmacro/2) — is a [`macro`](https://github.com/elixir-lang/elixir/blob/ee758f987b7e240754bf386b903b92d38fb02233/lib/elixir/lib/kernel.ex#L4169) itself. Therefore, all we need is to define our own macro, which will create from the construction above both spec and a function declaration.

Vamonos.

### Step 1. Understanding the Task

Let’s start with unveiling the AST that our not-yet-built macro would receive as a parameter.

```elixir
defmodule CustomSpec do
  defmacro defs(args, do: block) do
    IO.inspect(args)
    :ok
  end
end

defmodule CustomSpec.Test do
  import CustomSpec

  defs is_forty_two(n: integer) :: boolean do
    n == 42
  end
end
```

Wow, wow, not so fast. We immediately ran into _formatter_’s rebellion. She spat a ton of parentheses here and there and has the code reformatted in a way that makes everybody having soul cry. Let’s wean her off it. To do that we need to change the configuration file `.formatter.exs` in the following way

```elixir
[
  inputs: ["mix.exs", "{config,lib,test}/**/*.{ex,exs}"],
  export: [locals_without_parens: [defs: 2]]
]
```

Let's go back to our daily duties and see what `defs/2` gets in there. It should be noted that `IO.inspect/2` will be executed at the compilation stage (if you do not understand why, you probably should not tweak macros yet, but read a brilliant book [Metaprogramming Elixir](https://pragprog.com/book/cmelixir/metaprogramming-elixir) by Chris McCord.) Also, to prevent the compiler from swearing, we return`: ok` (macros must return the correct AST). So:

```elixir
{:"::", [line: 7],
 [
   {:is_forty_two, [line: 7], [[n: {:integer, [line: 7], nil}]]},
   {:boolean, [line: 7], nil}
 ]}
```

Yeah. The parser believes that the imperator here is `::`, gluing the function definition and the return type. The function definition also contains a list of parameters as `Keyword`, `parameter name → type`.

### Step 2. Fail Fast

Since we have decided to support only that syntax for starters, we would need to rewrite the definition of the macro `defs` to raise immediately if, for example, the return type is not specified.

```elixir
defmacro defs({:"::", _, [{fun, _, [args_spec]}, {ret_spec, _, nil}]}, do: block) do
```

Well, it i-i-i-i-i-is an _Implementation Time_.

### Step 3. Generation of Both Spec and Function Declaration

```elixir
defmodule CustomSpec do
  defmacro defs({:"::", _, [{fun, _, [args_spec]}, {ret_spec, _, nil}]}, do: block) do
    # function declaration arguments
    args = for {arg, _spec} <- args_spec, do: Macro.var(arg, nil)
    # spec declaration arguments
    args_spec = for {_arg, spec} <- args_spec, do: Macro.var(spec, nil)

    quote do
      @spec unquote(fun)(unquote_splicing(args_spec)) :: unquote(ret_spec)
      def unquote(fun)(unquote_splicing(args)) do
        unquote(block)
      end
    end
  end
end
```

The code is so evident, that I hesitate to add anything to it.

Let’s see how would `CustomSpec.Test.is_forty_two(42)` behave

```elixir
iex> CustomSpec.Test.is_forty_two 42
#⇒ true
iex> CustomSpec.Test.is_forty_two 43
#⇒ false
```

Well, it works. I would not dump the BEAM chunk here, bear with me. You are to believe that spec works as well.

### Step 4. Is That It?

Of course not. In the wild, one would have to properly handle invalid calls, to implement header definitions for functions with several different default parameters, to collect the spec more accurately (with variable names included,) to make sure that all argument names are different, and much more. But as the proof of concept it’s already good enough.

Also, one still may surprise colleagues with something like

```elixir
defmodule CustomSpec do
  defmacro __using__(_) do
    import Kernel, except: [def: 2]
    import CustomSpec

    defmacro def(args, do: block) do
      defs(args, do: block)
    end
  end

  ...
end
```

(Also `defs/2` i s to be modified to generate `Kernel.def` instead of `def`,) but take it for granted: you do not want to suprise mates in such a weird way. Even on April 1st.

Happy macroing!
