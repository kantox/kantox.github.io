---
layout: post
title: "Elixir Iteraptor :: Iterating Nested Terms Like I’m Five"
description: "The tiny library to iterate/map/filter deeply nested structures in Elixir"
category: hacking
tags:
  - elixir
  - tricks
  - tools
---

Elixir data structures are immutable. That’s great from the perspective of insurance your data is not damaged in some other unrelated scope, but that’s a bit annoying when you need to modify deeply nested structure.

We have a brilliant [`Access`](https://hexdocs.pm/elixir/Access.html) abstraction, that allows four main operations on deeply nested terms with helpers exported from `Kernel`:

- [`Kernel.get_in/2`](https://hexdocs.pm/elixir/Kernel.html#get_in/2)
- [`Kernel.put_in/{2,3}`](https://hexdocs.pm/elixir/Kernel.html#put_in/2)
- [`Kernel.update_in/{2,3}`](https://hexdocs.pm/elixir/Kernel.html#update_in/2)
- [`Kernel.pop_in/{1,2}`](https://hexdocs.pm/elixir/Kernel.html#pop_in/2)

the above is supposed to be used like shown below.

```elixir
# getting the value
iex> users = %{"john" => %{age: 27, mood: "👍"}, "meg" => %{age: 23}}

iex> get_in(users, ["john", :age])
#⇒ 27

iex> put_in(users, ["john", :age], 28)
#⇒ %{"john" => %{age: 28, mood: "👍"}, "meg" => %{age: 23}}

iex> update_in(users, ["john", :mood], & &1 + 1)
#⇒ %{"john" => %{age: 28, mood: "👍"}, "meg" => %{age: 23}}

iex> pop_in(users, ["john", :mood])
#⇒ {"👍", %{"john" => %{age: 27}, "meg" => %{age: 23}}}
```

It’s handy and it’s working in many cases, unless it does not. To use `Access` one should know the path to the target element, and it requires a considerable amount of boilerplate to deal with many nested terms (like removind all leafs having `nil` value, or shadowing all the private fields with stars.)

To cover the demand for easy nested terms traversal, `Iteraptor` package was created.

**TL;DR:**

- [source code](https://github.com/am-kantox/elixir-iteraptor);
- [documentation](https://hexdocs.pm/iteraptor/Iteraptor.html).

---

Iterating both maps and lists in Elixir is charming. One might chain iterators, map, reduce, filter, select, reject, zip... Everybody having at least eight hours of experience with Elixir has definitely seen (and even maybe written) something like this:

```ruby
~w|john jane jack joe|
|> Enum.map(&String.capitalize/1)
|> Enum.each(fn capitalized_name ->
     IO.puts "Hello, #{capitalized_name}!"
   end)
```

That is indeed handy. The things gets cumbersome when it comes to deeply nested structures, like a map having nested keywords, lists etc. The good example of that would be any configuration file, having nested subsections.

The amount of questions on Stack Overflow asking “how would I modify a nested structure” forced me to finally create this library. The implementation in _Elixir_ looks a bit more convoluted since everything is immutable and one cannot just traverse a structure down to leaves, modifying whatever needed in-place. The iteration-wide accumulator is required.

That is probably the only example I met in my life where mutability makes things easier. As a bonus the implementation of `bury/4` to store the value deeply inside a structure, creating the intermediate keys as necessary, was introduced. It behaves as a [proposed but rejected in ruby core](https://bugs.ruby-lang.org/issues/11747) `Hash#bury`.

Also, `

---

So, welcome the library that makes the iteration of any nested map/keyword/list
combination almost as easy as the natural Elixir `map` and `each`.

• [**Iteraptor**](https://github.com/am-kantox/elixir-iteraptor)

### Features

- [`Iteraptor.each/3`](https://hexdocs.pm/iteraptor/Iteraptor.html#each/3) to iterate a deeply nested map/list/keyword;
- [`Iteraptor.map/3`](https://hexdocs.pm/iteraptor/Iteraptor.html#map/3) to map a deeply nested map/list/keyword;
- [`Iteraptor.reduce/4`](https://hexdocs.pm/iteraptor/Iteraptor.html#reduce/4) to reduce a deeply nested map/list/keyword;
- [`Iteraptor.map_reduce/4`](https://hexdocs.pm/iteraptor/Iteraptor.html#map_reduce/4) to map and reduce a deeply nested map/list/keyword;
- [`Iteraptor.filter/3`](https://hexdocs.pm/iteraptor/Iteraptor.html#filter/3) to filter a deeply nested map/list/keyword;
- [`Iteraptor.jsonify/2`](https://hexdocs.pm/iteraptor/Iteraptor.html#jsonify/2) to prepare the term for JSON interchange; it basically converts keys to strings and keywords to maps because JSON encoders might have issues with serializing keywords;
- [`Iteraptor.Extras.bury/4`](https://hexdocs.pm/iteraptor/Iteraptor.Extras.html#bury/4) to store the value deeply inside nested term (the intermediate keys are created if necessary.)
- [`Iteraptor.to_flatmap/2`](https://hexdocs.pm/iteraptor/Iteraptor.html#to_flatmap/2) to flatten a deeply nested map/list/keyword into flatten map with concatenated keys;
- [`Iteraptor.from_flatmap/3`](https://hexdocs.pm/iteraptor/Iteraptor.html#from_flatmap/3) to “unveil”/“unflatten” the previously flattened map into nested structure;
- [`use Iteraptor.Iteraptable`](https://hexdocs.pm/iteraptor/Iteraptor.Iteraptable.html) to automagically implement `Enumerable` and `Collectable` protocols, as well as `Access` behaviour on the structure.

### Words are cheap, show me the code

#### Iterating, Mapping, Reducing

```elixir
# each
iex> %{a: %{b: %{c: 42}}} |> Iteraptor.each(&IO.inspect(&1, label: "each"), yield: :all)
# each: {[:a], %{b: %{c: 42}}}
# each: {[:a, :b], %{c: 42}}
# each: {[:a, :b, :c], 42}
%{a: %{b: %{c: 42}}}

# map
iex> %{a: %{b: %{c: 42}}} |> Iteraptor.map(fn {k, _} -> Enum.join(k) end)
%{a: %{b: %{c: "abc"}}}

iex> %{a: %{b: %{c: 42}, d: "some"}}
...> |> Iteraptor.map(fn
...>      {[_], _} = self -> self
...>      {[_, _], _} -> "********"
...>    end, yield: :all)
%{a: %{b: ""********", d: "some"}}

# reduce
iex> %{a: %{b: %{c: 42}}}
...> |> Iteraptor.reduce([], fn {k, _}, acc ->
...>      [Enum.join(k, "_") | acc]
...>    end, yield: :all)
...> |> :lists.reverse()
["a", "a_b", "a_b_c"]

# map-reduce
iex> %{a: %{b: %{c: 42}}}
...> |> Iteraptor.map_reduce([], fn
...>      {k, %{} = v}, acc -> {​{k, v}, [Enum.join(k, ".") | acc]}
...>      {k, v}, acc -> {​{k, v * 2}, [Enum.join(k, ".") <> "=" | acc]}
...>    end, yield: :all)
{​%{a: %{b: %{c: 42}}}, ["a.b.c=", "a.b", "a"]}

# filter
iex> %{a: %{b: 42, e: %{f: 3.14, c: 42}, d: %{c: 42}}, c: 42, d: 3.14}
...> |> Iteraptor.filter(fn {key, _} -> :c in key end, yield: :none)
%{a: %{e: %{c: 42}, d: %{c: 42}}, c: 42}
```

#### Flattening

```elixir
iex> %{a: %{b: %{c: 42, d: [nil, 42]}, e: [:f, 42]}}
...> |> Iteraptor.to_flatmap(delimiter: "_")
#⇒ %{"a_b_c" => 42, "a_b_d_0" => nil, "a_b_d_1" => 42, "a_e_0" => :f, "a_e_1" => 42}

iex> %{"a.b.c": 42, "a.b.d.0": nil, "a.b.d.1": 42, "a.e.0": :f, "a.e.1": 42}
...> |> Iteraptor.from_flatmap
#⇒ %{a: %{b: %{c: 42, d: [nil, 42]}, e: [:f, 42]}}
```

#### Extras

```elixir
iex> Iteraptor.jsonify([foo: [bar: [baz: :zoo], boo: 42]], values: true)
%{"foo" => %{"bar" => %{"baz" => "zoo"}, "boo" => 42}}

iex> Iteraptor.Extras.bury([foo: :bar], ~w|a b c d|a, 42)
[a: [b: [c: [d: 42]]], foo: :bar]
```

### In Details

#### Iterating

**`Iteraptor.each(term, fun/1, opts)`** — iterates the nested structure, yielding
the key and value. The returned from the function value is discarded.

- _function argument:_ **`{key, value}`** tuple
- _options_: **`yield: [:all, :maps, :lists, :none]`**, `:none` is the default
- _return value_: **`self`**

#### Mapping and Reducing

**`Iteraptor.map(term, fun/1, opts)`** — iterates the nested structure,
yielding the key and value. The value, returned from the block
should be either a single value or a `{key, value}` tuple.

- _function argument:_ **`{key, value}`** tuple
- _options_: **`yield: [:all, :maps, :lists, :none]`**, `:none` is the default
- _return value_: **`mapped`**

**`Iteraptor.reduce(term, fun/2, opts)`** — iterates the nested structure,
yielding the key and value. The value, returned from the block
should be an accumulator value.

- _function arguments:_ **`{key, value}, acc`** pair
- _options_: **`yield: [:all, :maps, :lists, :none]`**, `:none` is the default
- _return value_: **`accumulator`**

**`Iteraptor.map_reduce(term, fun/2, opts)`** — iterates the nested structure,
yielding the key and value. The value, returned from the block
should be a `{​{key, value}, acc}` value. The first element of this tuple is
used for mapping, the last—accumulating the result.

- _function arguments:_ **`{key, value}, acc`** pair
- _options_: **`yield: [:all, :maps, :lists, :none]`**, `:none` is the default
- _return value_: **`{mapped, accumulator}`** tuple

#### Filtering

**`Iteraptor.filter(term, filter/1, opts)`** — filters the structure
according to the value returned from each iteration (`true` to leave
the element, `false` to discard.)

- _function argument:_ **`{key, value}`** tuple
- _options_: **`yield: [:all, :maps, :lists, :none]`**, `:none` is the default
- _return value_: **`filtered`**

#### Flattening

**`Iteraptor.to_flatmap(term, opts)`** — flattens the structure into
the flatten map/keyword, concatenating keys with a delimiter.

- _options_: **`delimiter: binary(), into: term()`**,
  defaults: `delimiter: ".", into: %{}`
- _return value_: **`flattened`**

**`Iteraptor.from_flatmap(term, fun/1, opts)`** — de-flattens the structure from
the flattened map/keyword, splitting keys by a delimiter. An optional transformer
function might be called after the value is deflattened.

- _function argument:_ **`{key, value}`** tuple
- _options_: **`delimiter: binary(), into: term()`**,
  defaults: `delimiter: ".", into: %{}`
- _return value_: **`Map.t | Keyword.t | List.t`**

#### Extras

**`Iteraptor.jsonify(term, opts)`** — converts all the keys to binaries and all the keywords to maps.

- _options_: everything that is accesped by `Iteraptor.map/3`
- _return value_: the same term with all the keys as binaries and all the keywords converted to maps,

```elixir
iex> Iteraptor.jsonify([foo: [bar: [baz: :zoo], boo: 42]], values: true)
%{"foo" => %{"bar" => %{"baz" => "zoo"}, "boo" => 42}}
```

**`Iteraptor.Extras.bury/(term, key, value, opts)`** — puts the value under the deeply nested key, creating all the intermediate terms if needed.

- _term_: term to bury the value into
- _key_: key to bury the value under
- _value_: value to bury
- _options_: **`[into: :default | :map | :keyword]`** — map, keyword, or derive it from the parent keys (_default_)
- _return value_: the same term with the new value buried into.

```elixir
iex> Iteraptor.Extras.bury([foo: :bar], ~w|a b c d|a, 42)
[a: [b: [c: [d: 42]]], foo: :bar]

iex> Iteraptor.Extras.bury([foo: :bar], ~w|a b c d|a, 42, into: :map)
[a: %{b: %{c: %{d: 42}}}, foo: :bar]

iex> Iteraptor.Extras.bury(%{foo: :bar}, ~w|a b c d|a, 42, into: :keyword)
%{a: [b: [c: [d: 42]]], foo: :bar}
```

---

The source code is linked above, the package is available through
[`hex`](https://hex.pm/packages/iteraptor).

Happy iterapting!
