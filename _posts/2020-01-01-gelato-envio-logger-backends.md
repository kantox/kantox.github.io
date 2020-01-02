---
layout: post
title: "Monitoring Applications With Logger Backends"
description: "Introduces wider approach to creating and using Logger backends"
category: hacking
tags:
  - elixir
  - tricks
---

_Elixir_ derives logging infrastructure from _Erlang_. Starting with version _1.10_ which is to be released soon, _Elixir_ fully leverages new custom logging features happened to appear in _Erlang/OTP 21+_.

While _OTP_ provides the whole infrastructure to deliver logging events to subscribers, the _logging itself, in terms of storing / showing log events to the user_ is supposed to be implemented by the application. For that purpose, the `Logger.Backend` concept is introduced. The excerpt from the official documentation:

> `Logger` supports different backends where log messages are written to.
>
> The available backends by default are:
>
> * `:console` — logs messages to the console (enabled by default)
>
> Any developer can create their own `Logger` backend. Since `Logger` is an event manager powered by `:gen_event`, writing a new backend is a matter of creating an event handler, as described in the `:gen_event` documentation.
>
> The initial backends are loaded via the `:backends` configuration, which must be set before the `:logger` application is started.

---

Most common approach that gave a birth to many similar libraries all around is to create a `Logger.Backend` that understands and prints out _JSON_, and to attach some log consumer to the file produced. That way logs might reach any _log accumulating service_, which is usually a _NoSQL_ database, like _Elastic_.

We also store our logs in _Elastic_, but the modern way of mastering applications includes providing some metrics with log messages. The standard de facto for OTP applications would be [Telemetry](https://www.erlang-solutions.com/blog/introducing-telemetry.html), which is a _new open source project aiming at unifying and standardising how the libraries and applications on the BEAM are instrumented and monitored_.

_Telemetry_ is deadly simple, you call [`:telemetry.execute/2`](https://hexdocs.pm/telemetry/telemetry.html#execute-2) whenever in your application and the registered (_attached_ in their terminology) event listener will be called back. Also, there is an ability to attach [`Telemetry.Poller`](https://hexdocs.pm/telemetry_poller) to query the metrics periodically. The example in the article linked above suggests to call [`Logger.log/3`](https://hexdocs.pm/logger/Logger.html#log/3) from inside telemetry event handler.

### `Gelato`

The truth is I hate boilerplate code. I want everythign that might be done by compiler, scheduler and workers to be done without me even paying attention to it. For that I often pack the boilerplate into tiny libraries that hide all the boilerplate under their hood and expose clean interfaces to perform actions our application needs. That said, I wanted something that can be called like `report("message", payload)` to deliver logs, with telemetry data included, to our _Elastic_ storage.

Turns out, it was not too hard to accomplish.

We decided to use a custom `Logger.Backend` as an interface, so we could inject the desired functionality into _already existing projects_. Add a new logger backend to the existing project with `config :logger, backends: [Our.Fancy.Logger.Backend]`—and voilà—logs with telemetry attached are now being sent to _Elastic_.

That’s how [`Gelato`](https://hexdocs.pm/gelato/) library appeared. I know severe thoughtful true developers like libraries to be named in less esoteric way, but I am not a true developer by any mean. Bear with me. Although, _gelato_ (which is ice-cream in Italian btw,) is somewhat consonant to _elastic_.

The library is highly opinionated and it uses [_convention over configuration_](https://en.wikipedia.org/wiki/Convention_over_configuration) paradigm. It packs whatever one might need into a single _JSON_ and sends it to preconfigured _Elastic_ server via _HTTP_ request. It also attaches all the metadata it has an access to, like meaningful data retrieved with [`Process.info/1`](https://hexdocs.pm/elixir/master/Process.html?#info/1) etc.

To start using this library in the project, one should add the following to their `config/releases.exs` file:

```elixir
config :gelato,
  uri: "http://127.0.0.1:9200", # Elastic API endoint
  events: [:foo, :bar],         # attached telemetry events
  handler: :elastic             # or :console for tests

config :logger,
  backends: [Gelato.Logger.Backend],
  level: :info
```

Once added, any call to `Logger.log/3`, like the one below, will be passed through `telemetry` and sent to the configured _Elastic_ instance.

```elixir
Logger.info "foo",
  question: "why?",
  answer: 42,
  now: System.monotonic_time(:microsecond)
```

Also the library introduces the [`Gelato.bench/4`](https://hexdocs.pm/gelato/Gelato.html#bench/4) macro, that accepts a block and performs two calls to `Logger.log/3`; one before the block execution and another one right after.

---

`Gelato` implies the better interfacing of the projects by introducing [`Gelato.defdelegatelog/2`](https://hexdocs.pm/gelato/Gelato.html#defdelegatelog/2) macro, which is basically a composition of `Gelato.bench/4` _and_ [`Kernel.defdelegate/2`](https://hexdocs.pm/elixir/master/Kernel.html?#defdelegate/2). By using this macro, one might both extract all the interfaces of the project into a limited set of near-the-top modules _and_ make this calls logged and telemetried.

### `Envío.Log`

Another `Logger.Backend` implementation that was born in our tech torture chamber is [`Envío.Log`](https://hexdocs.pm/envio_log). It leverages [`Envío`](https://hexdocs.pm/envio) library to send the messages to a dedicated _Slack_ channel. It has it’s own `log_level` setting, which is usually set to either `:warn` or `:error` to prevent a _Slack_ channel from overflooding, and it purges all the calls to levels less that configured at compile time.

The typical configuration would look like:

```elixir
config :envio, :log,
  level: :warn,        # do not send :info to Slack
  process_info: false  # do not attach process info

config :logger, backends: [Envio.Log.Backend], level: :debug

config :envio, :backends, %{
  Envio.Slack => %{
    {Envio.Log.Publisher, :info} => [
      hook_url: {:system, "YOUR_SLACK_CHANNEL_API_ENDPOINT"}
    ]
  }
}
```

Once configured, all the calls to `Logger.{warn,error}/2` would be sent to the respective _Slack_ channel. This is extermely handly for monitoring the workflows in production in the real time.

Happy logging!
