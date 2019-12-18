---
layout: post
title: "StringNaming to call UTF8 by name"
description: "compile-time generated set of modules to ease an access to a predefined subset of UTF8 symbols"
category: hacking
tags:
  - tricks
  - elixir
---

## WTF UTF8?

There are many good tutorials on how Elixir deals with UTF8 and why it is the only
language so far that properly converts “José” string to capitals out of the box.
If you are unfamiliar with the subject, I would recommend the brilliant writing
[Unicode and UTF-8 Explained](https://www.bignerdranch.com/blog/unicode-and-utf-8-explained/)
by Nathan Long.

In a nutshell, Elixir’s [`String.upcase/1`](https://hexdocs.pm/elixir/String.html#upcase/1)
walks through the string, performs a normalization (for accents and other combined
symbols), and finally converts graphemes to their capital representation. This
process is fully automated and—which is more important—is always up-to-date,
since it reads the conversion rules directly from Consortium’s
[definition files](http://www.unicode.org/Public/UCD/latest/ucd/). Two used in
case conversion are `UnicodeData.txt` and `SpecialCasing.txt`, if you one’s curious.

In this post we’ll build the relatively same stuff to address fancy UTF8 symbols
by their names. The whole codebase contains 119 LOCs. In the end we’ll yield
a bundle of modules, providing functions returning the UTF8 symbol given it’s name
as a function name. The practical use of this package is beyond the scope of this
post, but I am positive there could be many.

## Bruteforce approach

So far, so good. Our goal is to end up with something like:

```elixir
iex|1 ▶ StringNaming.AnimalSymbols.monkey
"🐒"
```

The gracefully stolen from native Elixir [`unicode/properties.ex`](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/unicode/properties.ex) approach would be to:

— parse text file provided by Consortium;  
— prepare the data structure to walk through;  
— build the functions in compile time using meta programming features.

This is all mighty and will prevail, except it ain’t so.

There is no problem reading the file, as well as the data structure needed
is to be built without a glitch. The problem is that we need nested modules
to be produced on the fly. Look:

```elixir
iex|2 ▶ StringNaming.AnimalSymbols.<TAB>
Baby            Bactrian        Dromedary       Fox
Front           Hatching        Lady            Lion
Paw             Spiral          Tropical        Unicorn
Water           ant/0           bat/0           bird/0
blowfish/0      boar/0          bug/0           butterfly/0
cat/0           chicken/0       chipmunk/0      cow/0
crab/0          crocodile/0     deer/0          dog/0
dolphin/0       dragon/0        duck/0          eagle/0
elephant/0      fish/0          goat/0          gorilla/0
honeybee/0      horse/0         koala/0         leopard/0
lizard/0        monkey/0        mouse/0         octopus/0
owl/0           ox/0            penguin/0       pig/0
poodle/0        rabbit/0        ram/0           rat/0
rhinoceros/0    rooster/0       scorpion/0      shark/0
sheep/0         shrimp/0        snail/0         snake/0
spider/0        squid/0         tiger/0         turkey/0
turtle/0        whale/0
iex|2 ▶ StringNaming.AnimalSymbols.Baby.chick
"🐤"
```

From the above we see that there are nested modules along with plain functions
inside nearly each module. And, unluckily, Elixir does not allow to re-open
modules as Ruby does. So, we are to use recursion in our metaprogramming
adventure. Exciting?

### Reading the file

There is nothing special with reading the file: it’s plain text in very
simple format. Category names are prepended with `"@\t"` making it easy
to pattern match, codepoints are followed by their names: take and parse.

On that stage we simply collect everything in the list of tuples
`{code, name, category}`. E.g. for the monkey face above, this tuple would
be `{"1F412", "MONKEY", "AnimalSymbols"}`.

The boring code, in case anybody is curious, [may be found here](https://github.com/am-kantox/string_naming/blob/master/lib/string_naming.ex#L81-L97).

### Converting the list to nested map

Second stage is to convert everything to the nested map, so that later on
we could recursively iterate it. This is
[plain old good Elixir](https://github.com/am-kantox/string_naming/blob/master/lib/string_naming.ex#L99-L112)
as well: no tricks, no exciting shining ideas.

Codepoint is the leaf in each nested map.

### Building modules

That is the part the whole post was written for. Impatient readers might
just read [these 36 LOCs](https://github.com/am-kantox/string_naming/blob/master/lib/string_naming.ex#L1-L36),
for others we’ll walk through step by step.

The first pitfall is that we need to call to produce the nested modules from inside
the definition of the currently operated one. That said, we need to declare
the dedicated module to deal with that, otherwise scopes won’t allow us to do that.
BTW, we’ll `:code.delete` and `:code.purge` this helper module afterward.

The first step is to write a flat level iteration. That is relatively easy:

```elixir
def nesteds(nested, %{} = map) do
  Enum.each(map, fn
    {_key, code} when is_binary(code) -> :ok # leaf, skip it
    {k, v} ->
      mod = :lists.reverse([k | :lists.reverse(nested)])
      StringNaming.H.nested_module(mod, v)
  end)
end
```

The above just iterates the map and calls a producer for all the nested
modules. That simple. `StringNaming.H.nested_module/2` is where the deal
happens. First of all, we are to split functions (leaves) and modules (branches).
We could not prepare this in advance, since we had no clue at parsing stage
whether this would be a leaf or not.

```elixir
[funs, mods] = Enum.reduce(children, [%{}, %{}], fn
  {k, v}, [funs, mods] when is_binary(v) ->
    [Map.put(funs, k, v), mods]
  {k, v}, [funs, mods] ->
    [funs, Map.put(mods, k, v)]
end)
```

We consider binaries to be a codepoint value, and, hence, a leaf. Yes, I am aware
of [`Enum.split_with/2`](https://hexdocs.pm/elixir/Enum.html#split_with/2), but
here it’s simpler (and faster) to produce maps explicitly. Now we have two maps.
It’s time to rock!

The first—hacky and basically wrong—approach was to use
[`Code.eval_quoted/3`](https://hexdocs.pm/elixir/Code.html#eval_quoted/3),
since I could not figure out how to dynamically create a module inside other module:

```elixir
defmodule Module.concat(mod) do
  Enum.each(funs, fn {name, value} ->
    # name might be numeric, e.g. 1 ⇒ make it a proper atom here
    name = name
           |> String.replace(~r/\A(\d)/, "N_\\1")
           |> Macro.underscore
           |> String.to_atom
    # abstract syntax tree of
    # ★  def monkey, do: "🐒"
    # value is a codepoint
    ast = quote do
            def unquote(name)() do
              <<String.to_integer(unquote(value), 16)::utf8>>
            end
          end
    Code.eval_quoted(ast, [name: name, value: value], __ENV__)
  end)
  # TODO: def __all__
  StringNaming.H.nesteds(mod, mods) # call back for the nesteds
end
```

I have posted a question on SO and
[Dogbert helped](https://stackoverflow.com/a/44852119/2035262) me to make
the code clean with [`Module.create/3`](https://hexdocs.pm/elixir/Module.html#create/3):

```elixir
ast = for {name, value} <- funs do
  name = name |> String.replace(~r/\A(\d)/, "N_\\1") |> Macro.underscore |> String.to_atom
  quote do: def unquote(name)(), do: <<String.to_integer(unquote(value), 16)::utf8>>
end
# TODO: def __all__
Module.create(Module.concat(mod), ast, Macro.Env.location(__ENV__))
StringNaming.H.nesteds(mod, mods)
```

---

That is basically it. The only thing left is to implement `__MODULE__.__all__/0` function
to return a keyword list of all the functions available, with their values:

```elixir
  def __all__ do
    :functions
    |> __MODULE__.__info__()
    |> Enum.map(fn
        {:__all__, 0} -> nil
        {k, 0} -> {k, apply(__MODULE__, k, [])}
        _ -> nil
    end)
    |> Enum.filter(& &1)
  end
```

Now we just call

```elixir
StringNaming.H.nesteds(["String", "Naming"], names_tree)
```

on the top level, and the tree of modules is built under `StringNaming`
namespace. Enjoy:

```elixir
StringNaming.ChessSymbols.Black.Chess.king
"♚"
```

---

## Get the pill

There is not much more code in the package, besides the above,
but for those picky persons, we have
[string_naming @ github](https://github.com/am-kantox/string_naming), also the package
is [string_naming @ hex.pm](https://hex.pm/packages/string_naming).
