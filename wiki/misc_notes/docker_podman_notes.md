---
---

# Docker/Podman

## Dockerfile formatting

### Entrypoint

``ENTRYPOINT`` has multiple formats[^1].

It's important not to confuse Exec format and Shell format:
- Exec: ENTRYPOINT ["command", "arg1", "arg2"]
- Shell: ENTRYPOINT command arg1 arg2

The shell format is easier to type and may feel more natural, but it's a trap!
It does *not* allow for any additional arguments to pass through during
``docker/podman run``, and neither of those tools will provice a warning or
introspection about that.

[^1]: https://docs.docker.com/engine/reference/builder/#entrypoint

## Running images as containers

It's best practice with most containers to use an ``ENTRYPOINT`` which defines
a command which is long-running by nature. If several things need to be done,
it is common practice to use an ``entrypoint.sh`` script which waits for input,
such as using ``/bin/sh -c`` as the last command to execute.

If, for any reason, the ``ENTRYPOINT`` defined by a built image is not long
running, such as being a one-off command rather than being a server or some
other listening daemon, it may be best practice in management to run it with
``docker run --rm``, which will prevent commands-as-containers from spawning an
excess number of containers.
