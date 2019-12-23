---
layout: post
title: "Envío → PubSub inside OTP"
description: "Presenting Envío → PubSub helper for both local and distributed OTP applications"
category: hacking
tags:
  - elixir
  - erlang
---

![Sunrise in Torre Mapfre](/img/sunrise-titanic.jpg)

_OTP_ stands for _Open Telecom Platform_. It is a historical quirk; _Ericsson_ was a culpritin initiating the development of fault-tolerant software platform, hence _Telecom_. It turned out, _OTP_ as we know it has nothing to do with _Telecom_. Nowadays the title has the same connotations with its functionality, as apples are related to medium quality phones.

The main distinguishing feature of the OTP according to the authors, is fault tolerance. Not multithreading, not the actor model, not the rich pattern matching, even not transparent clustering and not hot code upgrades. Fault tolerance.

The virtual machine of erlang on the surface is very simple: there are a bunch of _“processes”_ (not system processes, erlang processes) with isolated memory, that can exchange messages. That’s it. As Joe Armstrong had written once:

> In my blog I argued that processes should behave pretty much like people. People have private memories and exchange data by message passing.  
> — [Why I don't like shared memory](http://armstrongonsoftware.blogspot.com/2006/09/why-i-dont-like-shared-memory.html)

The exchange of messages within _OTP_ is deadly simple. One process sends a message to another process (or to a group of other processes,) synchronously, or asynchronously. Note, that it is necessary to know who is to receive these messages. That is, the _manager_ of the exchange is the sender. But what if we wanted to send broadcast and to provide an opportunity to all interested processes to subscribe to this message?

Yes, we are naturally coming to a generic _PubSub_, but it is not available in _OTP_ out of the box. Luckily enough, all the bricks to build our own implementation are there. Let's get started.

### Embodiments

_Elixir_ includes a module [`Registry`](https://hexdocs.pm/elixir/master/Registry.html), which can be used as a [scaffold для pubsub](https://hexdocs.pm/elixir/master/Registry.html#module-using-as-a-pubsub). We are only to provide several lines of glue code, and the neat care for all workers. The only problem is `Registry` is _local_ and therefore it doesn’t support clustering. That is, in a distributed environment this beauty would not work.

OK, there is a distributed implementation [`Phoenix.PubSub`](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html), which comes with two implementations: [`Phoenix.PubSub.PG2`](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.PG2.html#content) and [`Phoenix.PubSub.Redis`](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html#module-direct-usage). Well, `Redis` is clearly the alien link in our chain; but `PG2` built on top of the erlang process groups [`pg2`](http://erlang.org/doc/man/pg2.html) it our saviour. It still requires a significant amount of code as a boilerplate in each project, though.

That said, we have everything to establish a convenient `PubSub` subscriptions in our app. Is it a time to open text editor? Not so fast. I personally don’t like to copy-paste code from one project to another. Instead everything that could be separated into kinda library gets extracted for reuse.

### Envío

Thus was born the package Envío. The talk is cheap, so I would start with showing the code.

#### Local broadcat → `Registry`

```elixir
defmodule MyApp.Sub do
  use Envio.Subscriber, channels: [{MyApp.Pub, :main}]

  def handle_envio(message, state) do
    # optionally call the default implementation
    {:noreply, state} = super(message, state)
    # handle it!
    IO.inspect({message, state}, label: "Received")
    # respond with `{:noreply, state}` as by contract
    {:noreply, state}
  end
end
```

This is all the code needed to produce fully functional subscriber. Just add `MyApp.Sub` to the supervision tree, and this process will begin receiving all the messages sent using functions in `MyApp.Pub` (which is in turn not too overwhelmed with the code as well.)

```elixir
defmodule MyApp.Pub do
  use Envio.Publisher, channel: :main

  def publish(channel, what), do: broadcast(channel, what)
  def publish(what), do: broadcast(what) # send to :main
end
```

#### Distributed newsletter → PG2

For distributed systems consisting of many nodes, the approach above would not work. We need to be able to subscribe to messages from other nodes, and `Registry` is not of any help here. Although we have `PG2` implementing the same behaviour that would serve all we need.

```elixir
defmodule MyApp.Pg2Sub do
  use Envio.Subscriber, channels: ["main"], manager: :phoenix_pub_sub

  def handle_envio(message, state) do
    {:noreply, state} = super(message, state)
    IO.inspect({message, state}, label: "Received")
    {:noreply, state}
  end
end
```

The only difference from the standalone code above would be `manager: :phoenix_pub_sub`  parameter that we pass to `use Envio.Subscriber` call (and `use Envio.Publisher` as well.) That would produce a module, backed by `:pg2` instead of local `Registry`. Now messages sent through this `Publisher` will be available on all nodes in the cluster.

### Application

_Envío_ supports the so-called backends. The package includes `Envio.Slack` that allows to utterly simplify sending messages to `Slack`. All that is required from the application to transparently start sending messages to `Slack` would be a channel configuration in the file `config/prod.exs`. Broadcast the message to the channel specified, _Envío_ will take care about the rest. Here is an example configuration:

```elixir
config :envio, :backends, %{
  Envio.Slack => %{
    {MyApp.Pub, :slack} => [
      hook_url: {:system, "SLACK_ENVIO_HOOK_URL"}
    ]
  }
}
```

Now all messages broadcasted with `MyApp.Pub.publish(:slack, %{foo: :bar})` will be delivered to the appropriate channel in `Slack`, nicely formatted. In order to stop sending messages to `Slack`, it’s enough to stop the process `Envio.Slack`. More examples (e.g. log in `IO`) can be found in the tests.

Well, why would you continue reading this? Simply try it yourself.

```elixir
def deps do
  [
    {:envio, "~> 0.8"}
  ]
end
```

Happy broadcasting!
