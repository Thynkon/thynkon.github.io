---
layout: post
title: Develop Phoenix applications using Docker
date: 2023-03-18 11:47 +0100
tags: [elixir, phoenix, docker]
---

## Context

For the last couple of months, I have been developing a few Elixir and Phoenix web applications. I have to admit I really enjoy Phoenix phylosophy.
For me, it offers the best Ruby on Rails features (quick development, code generators, etc...) but uses an explicit approach over the 'convention over configuration' ideology.

But that's out of the scope of this post. I will write a post in the next days about how I fell in love with Elixir.

So, back to the reason why I am writting this post. Recently, during a class about microservices,
I was asked to write a microservice using the language I loved the most.

Without hesitating, I chose Elixir. This was an opportunity to improve my Elixir/Phoenix skills as I have never writtern a [JSON RESTfull API](https://restfulapi.net).

It was also an opportuniy to share a new programming paradigm with my colleagues as most of them have only written code in `PHP`, `Java` and `C#`.

This was my way to contribute to the amazing `Elixir community`.

## Pre-requirements

In order to be able to follow this article, you need to have a working [Phoenix web application](https://www.phoenixframework.org).

You will also need to have [Docker](https://www.docker.com/) installed on your machine.

## Building the container

During my class, I was asked to create two diffent images for the project: a `development` and a `production` one.

Since most of tutorials/articles I found on the Internet only explained how to run `production-ready` applications on Docker containers, I had to create my own.

One of my requirements was to work on the project that was on the `host` without having to manually rebuild the container image. Thus, I had to use [bind volumes](https://docs.docker.com/storage/bind-mounts).

In case you dont know, Phoenix provides generators that make `production deployments` easy. You can pass `--docker` to `mix phx.gen.release` to [generate a docker container image](https://hexdocs.pm/phoenix/releases.html#containers)

Based on that file, I built the docker image you can find in the next section.

### Development

The Dockerfile below, uses an image provided by [Hex](https://hub.docker.com/r/hexpm/elixir).

As you can see, I install [Watchman](https://facebook.github.io/watchman/) which is required by [Phoenix Live Reloader](https://hexdocs.pm/phoenix_live_reload/Phoenix.LiveReloader.html). It analyses file modifications on a particular directory and perform some actions.

For a development environment, this is important as we do not have to manually reload the web server each time a file is modified.

Some extra packages like `procps`, `iproute2` and `lsof` are installed for debug purposes only. During the process of creating the development image, I encountered a few network problems regarding one of the best Elixir's features: [remote sessions](https://elixir-lang.org/getting-started/mix-otp/distributed-tasks.html).

In order to know what process where running on the container as well as what session token was being used, I had to use the [ps](https://man7.org/linux/man-pages/man1/ps.1.html) command combined with the only and only one [grep](https://man7.org/linux/man-pages/man1/grep.1.html).

**Dockerfile**

```docker
ARG ELIXIR_VERSION=1.14.2
ARG OTP_VERSION=25.1.2
ARG DEBIAN_VERSION=bullseye-20221004-slim

ARG BUILDER_IMAGE="hexpm/elixir:${ELIXIR_VERSION}-erlang-${OTP_VERSION}-debian-${DEBIAN_VERSION}"

FROM ${BUILDER_IMAGE} as builder

# install build dependencies
RUN apt-get update -y && apt-get install -y build-essential git curl vim bash watchman procps iproute2 lsof\
    && apt-get clean && rm -f /var/lib/apt/lists/*_*

# install hex + rebar
RUN mix local.hex --force && \
    mix local.rebar --force

# prepare build dir
ENV APP_HOME /app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME

ENV MIX_ENV="dev"

# Elixir remote session port
EXPOSE 4369
EXPOSE 4000

COPY ./entrypoint.sh /app
RUN chmod +x /app/entrypoint.sh

ENTRYPOINT ["/app/entrypoint.sh"]
```

This `Dockerfile` exposes two ports:

- `4000`: Phoenix web server
- `4369`: [Erlang Port Mapper Daemon](https://www.erlang.org/doc/man/epmd.html), a program that allows to use `remote sessions`.

**Note**: If you don't need to export the `4369` port in order to develop a web application in Phoenix using a container. It is a optional feature that can deeply increate your development experience.

If you have any experience working with Docker containers, you know that once you reach a certain level of complexity, building and running containers using the `docker` command can be painful.

Hence, we will use [docker-compose](https://docs.docker.com/compose/reference). As I mentioned before, the `host` directory where your application is stored is mounted on the container using a `bind volume`.

Also, a network is set so we can easily communicate with the container. Here, we set the ip address of the host to `172.40.0.1` and the container ip address to `172.40.0.2`.

Feel free to change and adapt those values according to your needs.

Finally, we tell Docker to execute the file called `entrypoint.sh` once the container has started.

**docker-compose.yml**

```yml
version: "3.3"
services:
  app:
    image: <YOUR_IMAGE_NAME>
    build:
      context: "."
    container_name: <YOUR_CONTAINER_NAME>
    networks:
      phoenix:
        ipv4_address: 172.40.0.2
    ports:
      - "4000:4000"
    environment:
      - <YOUR_ENV_VARIABLE>: ""
    volumes:
      - type: bind
        source: .
        target: /app
networks:
  phoenix:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.40.0.0/24
          gateway: 172.40.0.1
```

This is a simple shell script that installs and compiles dependencies as well as runs database migrations.

But, the last command is the most important one. It starts a remote session on the container named `docker@172.40.0.2` with a cookie named `my_cookie`.

The session's name is usually made up of the `hostname` and `ip address` of the remote machine. This allows us to identify/[list](https://hexdocs.pm/elixir/1.14.3/Node.html#list/0) remote nodes.

Without the session cookie, you will not be able to connect to the remote session.

And last but not least, we start the Phoenix web server.

**entrypoint.sh**

```sh
#!/bin/bash

set -e

# Ensure the app's dependencies are installed
mix deps.get

# Compile dependencies
mix deps.compile

# Create database and run migrations
mix ecto.create
mix ecto.migrate

# Launch Elixir's remote session
elixir --name docker@172.40.0.2 --cookie my_cookie -S mix phx.server

```

#### Running the docker image as a non-root user

At my class, I used a docker image as part of a CI/CD pipeline. All the tests and other tasks regarding the code quality
were run on the docker container.

And, in case you don't know, when you launch Phoenix's web server by typing `mix phx.server`, `mix` will actually
download and compile both the dependencies and the project code base. Since you build and run the docker image as root,
`_build` and `deps` will belong to `root`.

This happens because we are mounting the `hosts` directory to the docker container `/app/` directory.

When I tried to run my CI/CD pipeline, most of the tasks failled due to missing permissions. Thus, I decided
to run the docker image as an user other than `root`.

I knew that I wasn't the first person to have ever encounted this problem, so a quick research pointed me to this
excellent answer by [BMitch on Stackoverflow](https://stackoverflow.com/questions/44683119/dockerfile-replicate-the-host-user-uid-and-gid-to-the-image/44683248#44683248).

You can create an user for the docker image and set the `UID` and `GUID` of that user. If you are the only one using your
computer, chances are that your `UID` and `GUID` are both `1000`.

In case you want to set `UID` and `GUID` to a value other than `1000` you can pass it as a build option to docker.

Here are the final `Dockerfile` and `docker-compose.yml` files which allow you to run the docker container as a non-root user:

**Dockerfile**
```docker
ARG ELIXIR_VERSION=1.14.2
ARG OTP_VERSION=25.1.2
ARG DEBIAN_VERSION=bullseye-20221004-slim

ARG BUILDER_IMAGE="hexpm/elixir:${ELIXIR_VERSION}-erlang-${OTP_VERSION}-debian-${DEBIAN_VERSION}"

FROM ${BUILDER_IMAGE} as builder

ARG UNAME=bi
ARG UID=1000
ARG GID=1000
RUN groupadd -g $GID -o $UNAME
RUN useradd -m -u $UID -g $GID -o -s /bin/bash $UNAME

# install build dependencies
RUN apt-get update -y && apt-get install -y build-essential git curl vim bash watchman procps iproute2 lsof\
    && apt-get clean && rm -f /var/lib/apt/lists/*_*

USER $UNAME
# install hex + rebar
RUN mix local.hex --force && \
    mix local.rebar --force

# prepare build dir
USER root
ENV APP_HOME /app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME

RUN chown -R $UNAME:$UNAME $APP_HOME

COPY ./entrypoint.sh /app
RUN chmod +x /app/entrypoint.sh

ENTRYPOINT ["/app/entrypoint.sh"]

USER $UNAME

ENV MIX_ENV="dev"

# Elixir remote session port
EXPOSE 4369
EXPOSE 4000
```

**docker-compose.yml**
```yml
version: "3.3"
services:
  app:
    image: <YOUR_IMAGE_NAME>
    build:
      context: "."
      args:
        - "UID=${UID:-1000}"
        - "GID=${GID:-1000}"
    container_name: <YOUR_CONTAINER_NAME>
    networks:
      phoenix:
        ipv4_address: 172.40.0.2
    ports:
      - "4000:4000"
    environment:
      - <YOUR_ENV_VARIABLE>: ""
    volumes:
      - type: bind
        source: .
        target: /app
    user: 1000:1000
networks:
  phoenix:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.40.0.0/24
          gateway: 172.40.0.1
```

### Production

The `Dockerfile` of the production release was way easier to use as it is generated by [mix phx.release --docker](https://hexdocs.pm/phoenix/Mix.Tasks.Phx.Gen.Release.html#module-docker).

Based on that file, I only added the lines `77` and `78` which expose the ports used by the [EPMD](https://www.erlang.org/doc/man/epmd.html).

*Notice that the image's user is `root`. If you want to change that, do the same thing we did for the development image.*

**Dockerfile**

```docker
ARG ELIXIR_VERSION=1.14.2
ARG OTP_VERSION=25.1.2
ARG DEBIAN_VERSION=bullseye-20221004-slim

ARG BUILDER_IMAGE="hexpm/elixir:${ELIXIR_VERSION}-erlang-${OTP_VERSION}-debian-${DEBIAN_VERSION}"
ARG RUNNER_IMAGE="debian:${DEBIAN_VERSION}"

FROM ${BUILDER_IMAGE} as builder

# install build dependencies
RUN apt-get update -y && apt-get install -y build-essential git \
  && apt-get clean && rm -f /var/lib/apt/lists/*_*

# prepare build dir
WORKDIR /app

# install hex + rebar
RUN mix local.hex --force && \
  mix local.rebar --force

# set build ENV
ENV MIX_ENV="prod"

# install mix dependencies
COPY mix.exs mix.lock ./
RUN mix deps.get --only $MIX_ENV
RUN mkdir config

# copy compile-time config files before we compile dependencies
# to ensure any relevant config change will trigger the dependencies
# to be re-compiled.
COPY config/config.exs config/${MIX_ENV}.exs config/
RUN mix deps.compile

COPY priv priv

COPY lib lib

# Compile the release
RUN mix compile

# Changes to config/runtime.exs don't require recompiling the code
COPY config/runtime.exs config/

COPY rel rel
RUN mix release

# start a new build stage so that the final image will only contain
# the compiled release and other runtime necessities
FROM ${RUNNER_IMAGE}

RUN apt-get update -y && apt-get install -y libstdc++6 openssl libncurses5 locales lsof procps\
  && apt-get clean && rm -f /var/lib/apt/lists/*_*

# Set the locale
RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen

ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

WORKDIR "/app"
RUN chown nobody /app

ENV DATABASE_PATH=./<YOUR_PROJECT_NAME>.db

# set runner ENV
ENV MIX_ENV="prod"

# Only copy the final release from the build stage
COPY --from=builder --chown=nobody:root /app/_build/${MIX_ENV}/rel/business_intelligence ./

USER root

# Elixir remote session port
EXPOSE 4369
EXPOSE 4000

CMD ["/app/bin/server"]
```

**docker-compose*.yml***

```yml
version: "3.3"
services:
  app:
    image: <YOUR_IMAGE_NAME>
    build: .
    container_name: <YOUR_CONTAINER_NAME>
    networks:
      phoenix:
        ipv4_address: 172.40.0.2
    ports:
      - "4000:4000"
    environment:
      - YOUR_ENV_VARIABLE=

networks:
  phoenix:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.40.0.0/24
          gateway: 172.40.0.1

```

You also need to change the following file in your `Phoenix application`:

**env.sh.eex**
```elixir
# # Set the release to work across nodes.
# # RELEASE_DISTRIBUTION must be "sname" (local), "name" (distributed) or "none".
export RELEASE_DISTRIBUTION=name
export RELEASE_NODE="docker@172.40.0.2"
```

As per the [Elixir's documentation](https://elixir-lang.org/getting-started/mix-otp/config-and-releases.html#configuring-releases),
this file is generated and copied to each generated release.

The `RELEASE_NODE` environment variable is used to set the node's name. We need that in order to connect from a remote node.

#### Demo

To give a better idea of how [the observer](https://www.erlang.org/doc/man/observer.html) works I recorded a small demo where I connect to the remote session launched from the Docker container.

All you need to do in order to connect to a remote session is to type the following command:

```sh
iex --name <YOUR_USERNAME>@<YOUR_IP_ADDRESS> --cookie <YOUR_COOKIE>
```

Where:

- `<YOUR_USERNAME>` is the user of the host
- `<YOUR_IP_ADDRESS>` is the ip address of the gateway of the network created for the container. If we use the Dockerfile above as an example, it would be `172.40.0.1`.
- `<YOUR_COOKIE>` is the same cookie as the one defined in the `entrypoint.sh` file. In our case, it would be `my_cookie`.

<video style="max-width: 730px;" src="https://user-images.githubusercontent.com/35641748/207707176-2dd2acad-15be-4d5c-94a0-8c6912b9ed16.mp4" controls="controls"></video>
