---
layout: post
title: "Use Github CI for Elixir Projects"
description: "Quick and dirty howto on setting Github CI for Elixir projects"
category: hacking
tags:
  - elixir
  - erlang
---

![Welcome to Github CI](/img/filipines.jpg)

### Github CI

Github launched [actions](https://github.com/features/actions) which make it possible to perform CI without leaving the place where the code belongs. It is indeed quite handy. Once somebody pushes, or makes a pull request, or whatever (the list when to apply an action might be found in the [official documentation](https://help.github.com/en/articles/about-github-actions),) the build is started. Scheduled cron-like tasks are also supported.

One might produce pipelines of actions, named _workflows_. And all that is great, save for the documentation.

It took me almost an hour to figure out how to spawn a container with third-party services to test the application against. Here is what I have learned. Please note, that the official documentation is yet clumsy, incomplete and sometimes wrong.

The standard CI action uses the configuration files with the syntax _quite_ similar to the one used by [_CircleCI_](https://circleci.com/). It’s plain old good _YAML_, allowing to set up the target OS, environment, commands to execute, etc. Actions are _named_ what allows to refer to other actions and depend on them.

Also, the configuration allows to specify _services_. Services are to be run somewhere in the cloud and GH would map ports of the container to the ports these services expose, according to config. That part is feebly covered in the official documentation and even that what is covered contains errors.

Here is the working example of the configuration for the _Elixir_ project, requiring _RabbitMQ_ and _Redis_ services for testing.

```yaml
name: Tests for My Project

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest

    container:
      image: elixir:1.9.1-slim

    services:
      rabbitmq:
        image: rabbitmq
        ports:
        - 5672:5672
        env:
          RABBITMQ_USER: guest
          RABBITMQ_PASSWORD: guest
          RABBITMQ_VHOST: "/"
      redis:
        image: redis
        ports:
        - 6379:6379

    steps:
    - uses: actions/checkout@v1
    - name: Install Dependencies
      run: |
        MIX_ENV=ci mix local.rebar --force
        MIX_ENV=ci mix local.hex --force
        MIX_ENV=ci mix deps.get
    - name: Run All Tests
      run: |
        MIX_ENV=ci mix test
      env:
        RABBITMQ_HOST: rabbitmq
        RABBITMQ_PORT: $❴❴ job.services.rabbitmq.ports[5672] ❵❵
        REDIS_HOST: redis
        REDIS_PORT: $❴❴ job.services.redis.ports[6379] ❵❵
```

**NB** Curly brackets above should be normal ones, I uses these because my templating engine drives bonkers seeing two opening curlies, sorry for that.

As one might see, tests are to be run on _Ubuntu_, using _Elixir v1.9.1_. Services are described under the `services` key, and here is a trick. The port, the service port will be mapped to, is randomly chosen by the container engine in the runtime and stored in the _internal_ shell variable with a name `job.services.rabbitmq.ports[5672]`. `rabbitmq` here is the name of the service, as specified in this file in `services` section and `5672` is the original port. The internal variable has a syntax `$❴❴ foo ❵❵` and is being passed to the environment variable `RABBITMQ_PORT`. `RABBITMQ_HOST` there must be set to the _service name_. Now your application might read the environment variables as usual.

```elixir
import Config

config :my_app,
  rabbitmq: [
    host: System.get_env("RABBITMQ_HOST"),
    password: "guest",
    port: String.to_integer(System.get_env("RABBITMQ_PORT", "5672")),
    username: "guest",
    virtual_host: "/",
    x_message_ttl: "4000"
  ]
```

I have created a dedicated `mix` environment, called `:ci` to distinguish configuration for tests running in local vs. tests running there in the cloud.

---

Besides the CI I do run `dialyzer` on my sources. Since it’s running in a container, the task takes a while, because it needs to rebuild `plts` from the scratch every time. That is why I do it once a day, using `schedule` config.

```yaml
name: Dialyzer for My Project

on:
  schedule:
  - cron: "* 1 * * *"

jobs:
  build:
    runs-on: ubuntu-latest

    container:
      image: elixir:1.9.1-slim

    steps:
    - uses: actions/checkout@v1
    - name: Install Dependencies
      run: |
        MIX_ENV=ci mix local.rebar --force
        MIX_ENV=ci mix local.hex --force
        MIX_ENV=ci mix deps.get
    - name: Run All Tests
      run: |
        MIX_ENV=ci mix code_quality

```

where `code_quality` task is an alias declared in `mix.exs` as

```elixir
defp aliases do
  [
    code_quality: ["format", "credo --strict", "dialyzer"]
  ]
end
```

That is basically all we need to happily test the project with external dependencies in new Github Workflow.

Happy continiously intergrating!
