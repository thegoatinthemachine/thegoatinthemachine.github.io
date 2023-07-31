---
---

# Docker/Podman

## Dockerfile formatting

### Entrypoint

``ENTRYPOINT`` has multiple formats[1].

It's important not to confuse Exec format and Shell format:
- Exec: ENTRYPOINT ["command", "arg1", "arg2"]
- Shell: ENTRYPOINT command arg1 arg2

The shell format is easier to type and may feel more natural, but it's a trap!
It does *not* allow for any additional arguments to pass through during
``docker/podman run``, and neither of those tools will provice a warning or
introspection about that.

[1]: https://docs.docker.com/engine/reference/builder/#entrypoint

#### ...If your docker run command screws up

Check the entrypoint. Inspect the image, whatever, but it's a good idea before
rebuilding your entire container image from a different basis to first check
the entrypoint, and use a different `--entrypoint='["cmd","arg"]'` string

Same applies to podman.

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

## Running arbirary commands in live containers

TL;DR: use `podman exec`

Keeping in mind that a container is an instantiated/running image. If otherwise
unable to keep the container running, such as with a long-running service, use
the `--interactive` and `--detach` switches during `podman run` in order to
keep it alive for execution, such as:

```bash
podman run -di $image
```

Then podman exec can run the command and whitespace-separated args. Trying to
invoke it wrapped in quotes 'npx -y create-react-app' will treat the whole
literal string as the command to look for in the $PATH, so let it breath as a
whitespace separated list.

```bash
podman exec $container_name $command $command_args
```

## podman volume mounting issues on macOS

TL;DR: `podman machine stop && podman machine start`

Occasionally on macOS the podman VM  will cease to see the volume which is
bound to it. By default, it'll use the user's $HOME and mount it an equivalent
path on the VM. If you are expecting this and try mounting a volume from within
$HOME to some location in a container when it is not functioning, you'll get a
"file or directory doesn't exist". Unknown cause, maybe host macOS sleeping?
