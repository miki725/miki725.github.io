---
layout: post
title: "Reverse Engineering Docker Setup"
date: 2023-03-29
categories: [docker]
---

- TOC
  {:toc}

In this port Ill share my setup for simple CTF reverse engineering
which uses the following:

- [docker] for running processes to be reverse engineered
- [docker compose] for configuring container parameters
- [python] for the solver scripts which use excellent [pwntools]
- [poetry] for managing/locking python deps
- [gef] [gdb] plugin

Some familiarity with docker and docker compose will be necessary
in order to understand some of the scripts below.

The idea is as follows:

- build docker image which contains all necessities for debugging
- write python solver script
- run script in container
- pause script execution before it provides `stdin` to the spawned
  process
- attach to the script with `gdb` which allows to debug program

End result is that `gdb` is attached to a process however its
stdin/out/err will be controlled via `pwntools`

## Python Deps

First we need to create `pyproject.yml` file with all of our Python deps.
Easy way is via `init` command. If you do not have [poetry] installed,
you can use [pipx].

```sh
pipx install poetry
poetry init
```

You'll need to provide `pwntools` as a required dependency although
feel free to install any additional deps you may need. I highly
recommend `ptpython` and `pdbpp` for local Python debugging.

Here is my `pyproject.toml`:

```toml
# pyproject.toml

[tool.poetry]
name = "ctf"
version = "0.1.0"
description = ""
authors = ["Miroslav Shubernetskiy <ctf>"]
readme = "README.md"
packages = []

[tool.poetry.dependencies]
more-itertools = "^9.1.0"
pwntools = "^4.9.0"
python = "^3.10"
requests = "^2.28.2"

[tool.poetry.group.dev.dependencies]
mypy = "^0.991"
pdbpp = "^0.10.3"
ptpython = "^3.0.23"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

To generate `poetry.lock` file run:

```sh
poetry lock
```

## Docker image

The docker image is pretty simple as it installs all basic
deps for running the solver script and debugging the process.
Here is mine:

```dockerfile
# Dockerfile
FROM python:3.10

RUN apt-get update \
    && apt-get install -y \
        curl \
        gdb

RUN pip install poetry \
    && poetry config virtualenvs.create false

COPY pyproject.toml poetry.lock /

RUN poetry install

# install gef for gdb
RUN bash -c "$(curl -fsSL https://gef.blah.cat/sh)"
```

## Docker Compose

Starting docker container with all the necessary parameters sometimes is
annoying. Docker compose makes it much more simple. Here is the one I use:

```yaml
# docker-compose.yml
services:
  ctf:
    build: .
    volumes:
      - $PWD:$PWD
    working_dir: $PWD
    cap_add:
      - SYS_PTRACE
    security_opt:
      - seccomp=unconfined
    privileged: true
```

It will start the container with necessary permissions to use `gdb` to trace
container processes.

## Solver Script

At its core the requirement here is to:

- start process
- pause execution before some input is provided to the process
- attach `gdb` to the process
- resume script

```python
import pwn

pwn.context.log_level = "DEBUG"

s = pwn.process("binary", stdin=pwn.PTY)
pwn.pause()
s.send(b'\x00')
s.recvall()
```

To make the script a little more friendly for solving CTFs which
solve a specific binary, I usually name my solver script the same
as the binary with the `.py` extension which allows to automatically
pass the binary file from the solver script name:

```python
import pathlib

binary = f"./{pathlib.Path(__file__).with_suffix('').name}"
s = pwn.process(binary, stdin=pwn.PTY)
```

Also note that the solver script can work for both running local
binaries by using `pwn.process()` which will allow to attach `gdb` to
the spawned process or alternatively the script can work with remote
processes when the flag is only present on a remote machine:

```python
s = pwn.remote("ctf.server", 1234)
```

The rest of the `pwntools` API is the same between `process()` and
`remote()`.

## Running Solver With GDB

Now that we have a solver script we can spawn the binary with docker:

```sh
docker compose run -it --rm ctf \
   python binary.py
```

This will start the solver script and will pause before any input is
provided to the binary. Now in another tab we can attach `gdb` to the
spawned process within that container with a little shell magic:

```sh
docker exec -it $(docker compose ps --all --quiet)  \
    gdb \
        -p $(
            docker exec $(docker compose ps --all --quiet) \
                ps auxf \
                | grep python -A1 \
                | tail -n1 \
                | awk '{print $2}'
        ) \
        -ex 'finish'
```

This will automatically figure out the `pid` of the binary within
the container and will attach `gdb` to that binary within the same
container. This way you can just run this magic incantation and there is
no need to figure out any other parameters like `pid`, etc.

To break it down:

- `docker compose ps` will get the container id which is running
  the solver script
- `docker exec` executes a command within existing container
- `ps auxf` shows all processes in a parent-child order where the
  child processes appear after their parent processes. In this case
  this is important as we will be looking for a child process of
  `python` as we know the binary is spawned from python solver
  script therefore we know it will be a subprocess from `python`
- `grep python -A1` will find a line with `python` in it and
  will show one line after it as well
- `tail -n1` shows just the last line
- `aws '{print $2}'` will get the `pid` of the process
- `gdb -p <pid> -ex finish` will start `gdb` and immediately execute
  `finish` command inside `gdb`

After `gef` is running you can press anything in the first tab
which is running the solver script which will pass `stdin` to the
binary process and allow you to continue debugging in `gdb`.

## Mac

As this approach is running everything inside a docker container,
everything here works on a Mac which allows to RE from a Mac which will
allow you to edit the solver script in your favorite IDE and will also
allow you to run `x86_64` binaries even on M1 Mac if you build the
docker image for `linux/amd64` platform. You can do that by:

```sh
export COMPOSE_DOCKER_CLI_BUILD=1
export DOCKER_BUILDKIT=1
export DOCKER_DEFAULT_PLATFORM=linux/amd64
docker compose build
```

This will use [buildx] docker plugin to build another platform
docker container image. Buildx should be automatically installed
installed with [Docker for Mac].

## Illustration

[![asciicast](https://asciinema.org/a/2TtOidOtfBTqxUalRTc9vOmvA.svg)](https://asciinema.org/a/2TtOidOtfBTqxUalRTc9vOmvA)

[poetry]: https://python-poetry.org/docs/
[python]: https://www.python.org/
[docker]: https://docs.docker.com/desktop/
[docker compose]: https://docs.docker.com/compose/
[pwntools]: https://docs.pwntools.com/en/stable/
[gef]: https://github.com/hugsy/gef
[gdb]: https://www.sourceware.org/gdb/
[pipx]: https://pypa.github.io/pipx/
[buildx]: https://docs.docker.com/engine/reference/commandline/buildx/
[Docker for Mac]: https://docs.docker.com/desktop/install/mac-install/
