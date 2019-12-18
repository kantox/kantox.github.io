---
layout: post
title: "Adopting Property Testing in Elixir"
description: "Property testing gives way more assurance in many cases than other testing techniques"
category: hacking
tags:
  - elixir
  - tricks
  - tools
---

When I released the first robust version of [**`Iteraptor`**](https://hexdocs.pm/iteraptor), the _Elixir_ library for enumerations on steroids, I announced it on [_Elixir Forum_](https://elixirforum.com/t/iteraptor-iterating-nested-terms-like-i-m-five/13536), asking for how better can I test it. In the very first comment I received the reply that literally changed my understanding of how testing should be performed:

> This kind of functionality seems like the perfect fit for Property-based testing 😃

I never trusted tests much. It’s time to throw stones at me. I was always able to find better excuses for writing as few tests as possible. I rejected to test _third-party functionality_, _implementation details_, _internals of core library_, etc. Nah, seriously, either one trusts the framework in use, or it’d better by any mean to avoid using it at all.

All these countless questions on SO like _“how would I test that my rails controller receives `"foo"` when the fronend sends `"foo"`”_ drive me bonkers.—_“You don’t damn do it.”_ DHH already did. Trust him or switch to Wordpress.

Also, this is a human nature to test only what indeed passes. Mostly because if we knew it does not pass, we’d have it covered in the code so that it passes. Don’t tell me about TDD as well, it is truly better than covering the existing code with greenies, but still it is all about what we could imagine out of our head. The typical testsuite to cover the HTTP request would be:

- correct data → ✓ 200
- wrong data → ✗ 404

I am mature enough developer to produce a correct `if` statement, check the input validness and respond with either 200 or 404. I pretty fine understand also that this example is contrived, I am exaggerating and the real life is more complicated. True that. But we still fail to fortunetell the case where and by what our code is going to be screwed up. In the example above that would be something like � symbol coming from the user input. Malformed UTF8. Connection lost. Rats had the cable eaten. All that crap.

---

**We have to test what we cannot predict. Unfortunately, we are not able to predict what exactly we cannot predict.**

I am to repeat this again: testing good input, and bad input, and malformed input, and whatever else is an undoubted goodness. Everybody does it, and that’s great. That makes future code changes safer, simplifies the life and all that. That is not enough, though. Because see above.

---

Turning back to my library, it allows to `iterate`/`map`/`reduce`/`filter` deeply nested structures in _Elixir_, like maps, lists, keywords. I could come up with 50 examples or cumbersome deeply nested terms, no issue. I had exactly zero confidence that I have all the cases covered. And here property testing comes to the scene.

I have received this aforementioned advice to try property testing in the beginning of April. Two weeks later I attended [ElixirConfEU in Warsaw](http://www.elixirconf.eu/) and listened to the [brilliant talk by Andrea Leopardi](https://www.youtube.com/watch?v=p84DMv8TQuo) who is basically the core team member who [had it implemented for _Elixir_](https://andrealeopardi.com/posts/the-guts-of-a-property-testing-library/). It was a life changer.

Back in Barcelona I spent several hours implementing the [property testing for `Iteraptor`](https://github.com/am-kantox/elixir-iteraptor/blob/v1.2.1/test/property/iteraptor_test.exs). The whole file contains 75 LOCs and assures me my library works for many randomly generated nested structures of the max depth 25 (I performed the over-night run once I have it written—1K times with the depth 200.) Now I am 99% positive the declared functionality is robust.

---

It is indeed very easy to use. I will just drop a couple of examples here to show the general approach, for details please refer to Andrea’s post linked above.

First of all, you need to declare your possible data variants. I am iterating nested enumerables, hence I’ve had there

```elixir
import StreamData

  defmacrop aib, do: quote do:
    one_of([atom(:alphanumeric), integer(), binary()])

  defmacrop leaf_list, do: quote do: list_of(aib())
  # ...other enumerables...
  defmacrop leaf,
    do: quote do: one_of([leaf_list(), ...])

  defmacrop non_leaf_list, do: quote do: list_of(leaf())
  # ...other enumerables...
  defmacrop non_leaf,
    do: quote do: one_of([non_leaf_list(), ...])

  defmacrop maybe_leaf_list,
    do: quote do: list_of(one_of([leaf(), non_leaf()]))
```

That’s it. Now one simply needs to declare the expectations for any technically random data

```elixir
check all term <- maybe_leaf(), max_runs: 25 do
  reduced =
    Iteraptor.reduce(
      term, [], fn _, acc -> ["." | acc] end
    )
  mapped =
    term
    |> Iteraptor.map(fn _ -> "." end)
    |> Iteraptor.to_flatmap()
    |> Map.values()

  assert reduced == mapped
end
```

That’s it.
