---
layout: post
title: "Elixir Compilation Hooks"
description: "How to make custom macro `use` calls safer by checking the conditions met"
category: hacking
tags:
  - elixir
  - tricks
---

Elixir has a very sophisticated macro infrastructure. Also there is a wording you are to be immediately told when starting to deal with the language and getting excited about the power of macros. **“The first rule of using macros is you do not use macros.”** 

Sometimes they grudgingly append “unless you are in an urgent need and you know what you are doing.”

I agree one should not use macros in the very beggining of the journey. But once we dove deeply into this beatiful language, we all do extensively use them because macros allow us to drastically decrease an amount of boilerplate that might be needed _and_ provide the natural and very handy way to manipulate _AST_. _Phoenix_, _Ecto_, all the great huge libraries do heavily use macros.

The above is true for any multipurpose _library_/_package_. In my experience, regular projects usually do not require creating macros, or need only a few helpers to DRY. Libraries, contrary to the above, often consist of macros at ratio 80/20 to regular code.

I am not going to play sandbox here; if you wonder what macros are at all or how macros ever work in Elixir you’d better close this page immediately and read a brilliant [_Metaprogramming Elixir_](https://pragprog.com/book/cmelixir/metaprogramming-elixir) book by _Chris McCord_, the creator of _Phoenix Framework_. I am to only show some tricks to make your already existing macro ecosystem better.

---

Macros are purposedly stingy documented. This knife is too sharp to advertise it to toddlers.

Basically, macros are called back by the compiler when the external code calls `use MyLib` and our module `MyLib` implements `__using__/1` callback macro. If the above sounds cumbersome, please, stop bearing with me now and read the book I mentioned above instead.

`__using__` callback accepts an argument, so that the library owner might allow users to pass some parameters to it. Here is an example from one of my internal projects that uses a macro call with parameters:

```elixir
defmodule User do
  use MyApp.ActiveRecord,
    repo: MyApp.Repo,
    roles: ~w|supervisor client subscriber|,
    preload: ~w|setting companies|a
```

The keyword parameter will be passed to `MyApp.ActiveRecord.__using__/1` and there I deal with it.

---

Sometimes we want to restrict macro usage to some subset of modules (e. g. to allow using it in _structs_ only.) The explicit check inside the implementation of `__using__/1` won’t work, because at this moment the _currently being compiled_ module does not have an access to it’s `__ENV__` (and the latter is not complete by any mean.) So usually one wants somewhat check _after_ the compilation is done.

No issue, there are two [module attributes](https://hexdocs.pm/elixir/Module.html#module-module-attributes) designed explicitly for that purpose. Welcome, [_Compile Callbacks_](https://hexdocs.pm/elixir/Module.html#module-compile-callbacks)!

An excerpt from the docs:

> **`@after_compile`**
> A hook that will be invoked right after the current module is compiled.
>
  Accepts a module or a {module, function_name} tuple. The function must take two arguments: the module environment and its bytecode. When just a module is provided, the function is assumed to be `__after_compile__/2`.
>
  Callbacks registered first will run last.
>
    defmodule MyModule do
      @after_compile __MODULE__
      def __after_compile__(env, _bytecode) do
        IO.inspect env
      end
    end

I strongly encourage to never inject `__after_compile__/2` directly into generated code since it might lead to clashes with end-user intents (they might want to use their own compile callbacks.) Define a function somewhere inside your `MyLib.Helpers` or like and pass a tuple to `@after_compile`:

```elixir
quote location: :keep do
  @after_compile({MyLib.Helpers, :after_mymodule_callback})
end
```

This callback will be called immediately after the respective module that uses our library is compiled, receiving two parameters, the `__ENV__` struct and the bytecode of the compiled module. The latter is rarely used by mere mortals; the former provides everything we need. Below is the example of how do I defend from attempts to [`use Iteraptable`](https://hexdocs.pm/iteraptor/Iteraptor.Iteraptable.html) in non-structs by calling `__struct__` on the compiled module and delegating to Elixir core the right to raise a readable message when the module has no such field defined:

```elixir
def struct_checker(env, _bytecode), do: env.module.__struct__
```

The above will `raise` if the compiled module is not a struct. Of course, the code might be way more complex, but the core idea is whether your _used_ module expects something from the module that has it used, implement the `@after_compile` callback and damn raise unless all the prerequisites are met.

Happy compiling!
