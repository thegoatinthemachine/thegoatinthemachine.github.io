---
title: Jekyll build details
layout: post
excerpt_separator: <!-- more -->
---

It turns out that configure pages was mostly bust. I don't need it whatsoever
for building and testing jekyll with all the dependencies and plugins that
github-pages uses/allows.

So I moved on to actions-jekyll-build-pages. This is where the juicy stuff is,
as far as I can tell.

<!-- more -->

## actions/jekyll-build-pages

Because I can't be satisfied with knowing how I'm meant to interact with
something, I have a desire to know why and how it works. Again, everything here
is open source, so I started poking around the
[repository][gh_actions_jekyll_build_pages]. Ruby is largely unfamiliar to me,
so it took some adjustment to read everything going on, and understand what's
up with Gemfiles.

[gh_actions_jekyll_build_pages]: https://github.com/actions/jekyll-build-pages

### very special episode

I learned that a "gem" is a cutesy name for a ruby module/package which can be
handled programmatically by the ``gem`` ruby-environment package manager, and
the ``bundle`` dependency-resolver. Honestly I think this confuses the matter,
that the package shares the name of the package manager. To their credit, the
Gemfile format that ``bundle`` reads does require that the repository be
specified as a source.

I have opinions about this, TL;DR: that's silly, and makes adoption more
difficult. Any kind of cutesy clever crap like that which ignores commonly
established naming schemes in favor of one's own just throws a wrench in the
works. Interpreted languages having their own common package/module/library
repository used from a package manager is nothing new, such as CPAN, npm, pip,
etc. In fooling with ``callGraph`` (see previous entries), I needed to also
fool with CPAN, and I ended up resolving that by using cpanminus for zeroconf
quick setup.

### non-sequiter package management sidenote

As a sidenote, while I'm thinking of languages and their individual package
managers: these are usually platform-independent, and many of their libraries
are *not* found in larger package manager repositories, but only in the
language-specific repository. While *nix distro package managers are capable of
handling, theoretically, any kind of file that can have its installation image
standardized, the libraries one is likely to find are almost certainly going to
be for C and C++. Likewise, then, as ruby/gem and perl/CPAN and python/pip,
etc, deal with environments *specifically for* their own language, *nix can be
read as an environment specifically for C and friends. Certainly not a novel
thought, but I think it's intriguing. This may change as increasingly many
binaries for go and rust are shipped, creating demand for a healthy environment
for them.

Interestingly to me, there is pretty bad discoverability and availability of a
centralized C language repository, by comparison. Any given *nix package
manager worth its salt is likely to be able to connect to a repo which lists
many dozens of mature standard libraries of the kind shared by many individual
programs -- that's a large part of what lets linux programs take up so little
space by comparison with Windows or even macOS programs. Even with their own
dynamically loaded libraries, it's rare for any given several Windows or macOS
programs to share more than the barest dynamic libraries provided by the OS or
otherwise redistributable, and then many of those programs will just distribute
their own copy of, e.g, the vcredist files.

That being the case, individual distro repositories listing some well-known
standard C libraries does not a discoverable language
[repository][so_qst_language_repos] make.

[so_qst_language_repos]: https://stackoverflow.com/questions/1693529/list-of-top-repositories-by-programming-language

### jekyll build-pages functions

This action is basically packaged as a docker image ready-to-use. From what I
can tell, it has the following functions:

1. Slurp up inputs, which should have been defined earlier by either pages.yml
   or its own action.yml that the action runner would call as defined in
   pages.yml
2. Get default inputs that the github action runner provides (``act`` also does
   this, which took some digging), use those which are necessary for
   whitelisted plugins.
3. Run the ``github-pages`` ruby gem as a wrapper around jekyll to compile from
   the source to destination. This also, in the ``docker build`` process for
   this image, grabs all the enabled and whitelisted dependencies, and sets the
   configuration between their defaults, the user-provided inputs, and the
   configs they overwrite on top of the user's inputs.

I gathered most of that from ``entrypoint.sh``, and from inspecting
``Dockerfile``, and the ``Gemfile``. It's build systems all the way down!

Their ``Dockerfile`` defines an image built from a slim ruby base layer,
updated, copying in the ``Gemfile``, and installing the required gems and their
dependencies. Well, so that got me curious, in the ``Gemfile`` it asks for a
gem by name of github-pages, which is hosted on rubygems.org

### github-pages gem

The github pages [gem][gh_gh_pages_gem] is interesting. How they set
dependencies is a little convoluted, and relies on that ``gem`` will also do
some evaluation of actual ruby source when referenced. The vast bulk of the
package, by line count, is just defining dependencies and locking their
versions. All this information is reflected in some state by their
[webpage][gh_pages_versions] which similarly lists versions. I *could* use that
information if I really wanted to, and I certainly do not. The point is not to
develop a new image, the point is to make portable their work and use it
myself.

[gh_gh_pages_gem]: https://github.com/github/pages-gem
[gh_pages_versions]: https://pages.github.com/versions/

Ultimately, this gem is designed to be run standalone if desired, and includes
a standalone ``Dockerfile`` for its own image, which is available at github's
package repository, [ghcr][gh_gh_pages_gem_docker_image]. It is unclear to me
*why* exactly the actions/jekyll-build team went with building a new docker
image, building in a ruby container from their own ``Gemfile``. Left hand <-?->
right hand? I don't care that much, I only care which one I'm using, really.

[gh_gh_pages_gem_docker_image]: https://github.com/github/pages-gem/pkgs/container/pages-gem

Since the jekyll-build image is doing more legwork on my behalf, and
additionally defines dependencies in its ``Gemfile`` which are missing from the
github-pages gem (reason unknown), I'm moving forward with that.

## actions runner environment variables, and those actually used by the build container

It took some digging as to what exactly was going on here. When I was examining
the log files from ``act --verbose >>act.log 2>&1``, I was looking for where
and how environment variables were passed to the docker containers. The various
action.yml files clearly define parameters which would need to be delivered as
environment variables, after all. I was, heretofore, unfamiliar with how
exactly docker inserted environment variables.  At the command line this is
just ``docker -e ENV_VAR ...``, but the docker engine has an API. This, at
first a shock to me, is just straight up a REST API accessible with standard
HTTP interaction. On further examination, it does make sense, given that the
docker daemon and the CLI primarily communicate over a socket, typically at
``/var/run/docker.sock``. Anyway, the decision of the ``act`` team to use go
makes more sense, now, since the primary library they're interacting with is
the docker engine SDK, for which two official libraries are available: go and
python.

The go docker SDK defines a [type][gopkg_type_ExecConfig] which includes the
Env property, itself an array of strings. Near as I can tell, ``act`` uses that
in order to actually launch the docker image with all the environment
variables, some of which are defined as a default, and some of which are pulled
during runtime based on arguments and environment. I am unfamiliar enough with
go, and there are enough layers of indirection, that I actually can't tell
where this type gets used for an API call in the main path of execution. That's
rather besides the point, though. The main thing I was most interested in was
this [block][gh_nektos_act_withGithubEnv]. From what I can tell, very little,
if any, of that environment is necessary for me to replicate, but it explains
what I was seeing in the log file.

[gopkg_type_ExecConfig]: https://pkg.go.dev/github.com/docker/docker@v20.10.18+incompatible/api/types#ExecConfig
[gh_nektos_act_withGithubEnv]: https://github.com/nektos/act/blob/e1b906813e9053c57ef1c9f7d535532cd7924602/pkg/runner/run_context.go#L560-L592

### inspecting the docker container for environment variables in use

I want to poke around the ruby source and see which of the environment
variables are actually required, moreover, for which jekyll plugins.

I could go through the trouble of running the bundler and etc locally based on
the ``Gemfile`` that's provided as part of this docker image. But I don't need
to, nor do I need to futz with mangling or un-mangling my local ruby
environment: the container image would by definition already have all I need.
Moreover, the container should have the most critical environment variables
defined, absent of course the ones provided by the action runner when started
outside of that context.

The main container I have an interest in at the moment is jekyll-build-pages.
Alas, that has an entrypoint which directly calls the scripts, so if I try to
``docker run -it -d ghcr.io/actions/jekyll-build-pages``, I get nothing,
because the main ruby script errors out for lack of inputs. Again, glory to
open source, since I can make a quick edit to ``Dockerfile``, or to the
``entrypoint.sh`` script.

``act`` thankfully pulls all this by way of git clone or some equivalent, so I
have the git worktree available for me to work from here. More to the point, I
can munge up whatever I need and reset it with ``git reset --hard``.

An ENTRYPOINT will always run, and in this case, I do want ``entrypoint.sh`` to
run, since it sets several environment variables I want to inspect. The final
command, on [line 37][gh_actions_jekyll_build_entrypoint_L37], is what
actually calls the ruby script. However, even just removing that would not
allow me to let the script run and interactively futz with the container; the
bash script would immediately exit, which is where the container would stop its
process, and I'd be out of luck.

[gh_actions_jekyll_build_entrypoint_L37]: https://github.com/actions/jekyll-build-pages/blob/9c76bbb6c7b6fdbfde12014ed64206062b8eeac6/entrypoint.sh#L37

### hacking on the docker image to make it interactive

Docker's best-practices section about
[ENTRYPOINT][dockerdocs_best_practice_ENTRYPOINT] thankfully provides an
example of how to design a helper script to allow it to be interactive:

```bash
exec "$@"
```

Either directive of ``CMD`` or ``ENTRYPOINT`` can define an initial executable
and its arguments that run when a container starts. Similarly, only one of each
can exist in a ``Dockerfile``. The main difference is that ``CMD``, when used
alongside ``ENTRYPOINT``, provides *parameters* to whatever executable is
defined by ``ENTRYPOINT``. Either of these directives can be overwritten at the
command line:

- ``CMD`` provides a *default* which is overwritten by the optional parameters
  following the image when using ``docker run [options] IMAGE [command]``.
- ``ENTRYPOINT`` will always run, and will not be overwritten without a special
  argument of the form ``docker run --entrypoint="" ...``

So a helper script, as we have here, can have its last line defined as that
command to execute all arguments, and be run with an interactive shell by way
of

```bash
docker run -it IMAGE /bin/bash
```

I happen to know that this works because I was fooling around with it earlier.
If we needed to make sure that the command we're looking for actually exists,
we could export the filesystem of the image and read from that. Per this hint
on [stackoverflow][so_docker_export_tar]:

```bash
docker export $(docker ps -lq) | tar tf - | less
```

The shell-substitution command there, ``$(docker ps -lq)``, grabs just the id
of the most recently used container image. Lo and behold, ``/bin/grep`` also
exists in this image.

Per the various exports in ``entrypoint.sh``, then:

```bash
grep -e JEKYLL_ENV \
-e JEKYLL_GITHUB_TOKEN \
-e PAGES_REPO_NWO \
-e JEKYLL_BUILD_REVISION \
-e etc \
-r /usr/ \
--color=always | less -R
```

gives me nicely colored output I can page through.

[dockerdocs_best_practice_ENTRYPOINT]: https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#entrypoint
[so_docker_export_tar]: https://stackoverflow.com/a/46525825

### The exports from entrypoint.sh

- JEKYLL_ENV: set between 'production' or 'development', changes the behavior
  of some themes, of jekyll itself.
- JEKYLL_BUILD_REVISION: used by the
  [jekyll-github-metadata][gh_jekyll_github_metadata] plugin
- PAGES_REPO_NWO: Also used by the jekyll-github-metadata plugin
- JEKYLL_GITHUB_TOKEN: used by
	- jekyll-github-metadata plugin
	- [jekyll-gist][gh_jekyll_gist] plugin

The [jekyll-github-metadata][gh_jekyll_github_metadata] plugin wrangles
information accessible through liquid, by way of
[site.github][gh_jekyll_github_metadata_site.github]. PAGES_REPO_NWO -> "name
with owner".

[gh_jekyll_github_metadata]: https://github.com/jekyll/github-metadata
[gh_jekyll_github_metadata_site.github]: https://github.com/jekyll/github-metadata/blob/main/docs/site.github.md

The [jekyll-gist][gh_jekyll_gist] plugin extends liquid with a gist tag that
fetches the content of a github gist.

[gh_jekyll_gist]: https://github.com/jekyll/jekyll-gist

JEKYLL_ENV is probably the only one of those of real interest to me. I don't
know how much use I really have for the github metadata, or for the gist
plugin. The only function from the github metadata I might want, which is
pulling the most recent *public* repositories I've worked on, is also
accessible without authenticating, just making an API call to api.github.com

I think that's as far as I care to dig into this topic today.
