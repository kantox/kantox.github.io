---
layout: post
title: "Unveil Erlang Code of Your Elixir Project"
description: "How to see what erlang code was compiled to run your Elixir project"
category: hacking
tags:
  - tricks
  - elixir
  - erlang
---

## Elixir is Erlang in a nutshell

> `Elixir` leverages the Erlang VM [...]  
<small>[elixir-lang.org](https://elixir-lang.org/)</small>

What does mean “leverages”? It means Elixir is compiled into BEAMs, and the latter are
executed in Erlang VM. Elixir brings a lot of goodness, like arguable better syntax and
extensive macro support, but after all it’s the language on top of Erlang.

BEAMs are known to support decompilation back into Erlang code (providing they were
compiled with `debug_info` compiler option on.)

So, let’s configure our project to support decompilation back into Erlang code
in `MIX_ENV=dev` environment. That is relatively easy and might be helpful in some cases:
both educational and “wtf–investigations.” To do so we’ll need three things:

## Update your `config/dev.exs` to enable `debug_info`

Put this line in the very top of your `config/dev.exs` file:

```elixir
# https://hexdocs.pm/elixir/Code.html#compiler_options/1
Code.compiler_options(debug_info: true)
```

According to the [official documentation](https://hexdocs.pm/elixir/Code.html#compiler_options/1),

> `:debug_true` option is set to `true` to retain debug information in the compiled
module; this allows a developer to reconstruct the original source code, `false` by default.  
[...]  
These options are global since they are stored by Elixir’s Code Server.

## Prepare the Elixir BEAMs to be available

Before we are to create the script itself, let’s discover what do we need to support
Elixir code decompilation. Right: Elixir’s core compiled BEAMs themselves,
otherwise we would hardly decompile Elixir’s own goodness, like comprehensions etc.

I have no idea whether it’s possible with a standard Elixir distribution, but lickily
I nevertheless use trunk version (managed with [`exenv`](https://github.com/mururu/exenv)
version manager.) So, install `exenv` unless you have already done it, download
the source code of Elixir somewhere and compile it:

```bash
cd /usr/local/src && \
  git clone https://github.com/elixir-lang/elixir.git && \
  cd elixir && \
  make clean test
```

If everything went good, teach your `exenv` to use trunk version:

```bash
cd ~/.exenv/versions && \
  ln -s /usr/local/src/elixir trunk && \
  cd PATH_TO_MY_ELIXIR_PROJECT && \
  exenv local trunk
```

Check we are all set:

```bash
$ iex
Erlang/OTP 20 [erts-9.0] [source] [64-bit] [...]

Interactive Elixir (1.6.0-dev) - press Ctrl+C to exit (type h() ENTER for help)
iex|1 ▶ 
```

## Create a script to unveil the code

Copy the following lines into `/usr/local/bin/delixir`:

```erlang
#!/usr/bin/env escript
% -*- mode: erlang -*-

main([BeamFile]) ->
  Dir = "/usr/local/src/elixir/lib/elixir/ebin",
  code:add_patha(Dir),
  {ok,{_,[{abstract_code,{_,AC}}]}} = beam_lib:chunks(BeamFile,[abstract_code]),
  io:fwrite("~s~n", [erl_prettypr:format(erl_syntax:form_list(AC))]).
```

Make the file executable with `chmod +x /usr/local/bin/delixir` and you are all set.
Let’s check how it works:

```bash
delixir _build/dev/lib/my_app/ebin/Elixir.MyApp.beam
```

You should see something like:

```erlang
-file("/home/user/proj/my_app/lib/"
      "my_app.ex",
      1).
-module('Elixir.MyApp').
-compile(no_auto_import).
-behaviour('Elixir.GenServer').
-export(['__info__'/1, ...]).
-spec '__info__'(attributes | compile | exports |
		 functions | macros | md5 | module) -> atom() |
						       [{atom(), any()} |
							{atom(), byte(),
							 integer()}].
'__info__'(functions) ->
```

etc. This is an Erlang code that is basically exactly what Elixir was 
“transpiled” into before compilation. Or, better to say, this is how
Erlang virtual machine sees your Elixir code.
