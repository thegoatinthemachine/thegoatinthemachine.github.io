---
title: act for local development
---

## Using ``act`` for local development of the site

``act`` can be installed in both manjaro and macOS from their package
management. In manjaro, the default-installed tool ``pacman`` will work to pull
from the AUR. In macOS, ``brew`` will do the trick. Docker is a dependency.

But first

### Getting the correct github actions

the site needs to run on github actions explicitly. Github-pages started doing
this [transparently][gh_blog_pages_defaults_actions], in the background, in
August 2022. They [detailed][gh_blog_pages_actions_beta] some of this in July.
Hooray, that's easy to set up and because it was already doing this
transparently, using the defaults *should work*. By the time this post is live
on the site, this will already be true.

[gh_blog_pages_defaults_actions]: https://github.blog/2022-08-10-github-pages-now-uses-actions-by-default/
[gh_blog_pages_actions_beta]: https://github.blog/changelog/2022-07-27-github-pages-custom-github-actions-workflows-beta/

### A breakdown of what that does.

At time of writing, the version of github actions workflow at
.github/workflows/pages.yml does the following:

1. constructs build environment for "runner"s
2. checkout the codebase in the runner
3. configure jekyll in the runner
4. builds with jekyll and the other github-pages dependencies
5. tarballs and uploads the built artifacts to a server
6. unpacks and deploys the tarball to github-pages

### Resolving build errors

I set the repo to use the appropriate github action workflow, which by
definition involved copying their provided file to .github/workflows/pages.yml

``act`` will... act on this.

Running it the first couple times provides an error when it's running the build
step, configure-pages:

```bash
[Deploy Jekyll with GitHub Pages dependencies preinstalled/build]
[''] ⭐ Run Main Setup Pages
['']   🐳  docker cp src=/root/.cache/act/actions-configure-pages@v2/ dst=/var/run/act/actions/actions-configure-pages@v2/
[''] close /tmp/act1858139433: file already closed
['']   🐳  docker exec cmd=[node /var/run/act/actions/actions-configure-pages@v2/dist/index.js] user= workdir=
['']   💬  ::debug::all variables are set
| ::warning ::Get Pages site failed
| ::error ::Create Pages site failed
['']   ❗  ::error::AxiosError: Request failed with status code 401
['']   ❌  Failure - Main Setup Pages
[''] exitcode '1': failure
```

(Contents of brackets were all ditto the first, unnecessary visual clutter
removed) The resolution to this was plain as day in the readme at the act
[repository][gh_nektos_act]. I examined the action file, which the log tells me
should be in ~/.cache/act/actions-configure-pages@v2/action.yml and found under
inputs at line 14:

```yml
  token:
    description: 'GitHub token'
    default: ${{ github.token }}
    required: true
```

And so it is, I need to provide a Personal Access Token generated on github. I
created a dummy token to test this procedure, and discovered that for this to
work, at least sufficient permissions for workflows are necessary.

[gh_nektos_act]: https://github.com/nektos/act

The next problem was that the artifact upload process was not finding an
environment variable it needed:

```bash
[''] ⭐ Run Main Upload artifact
['']   🐳  docker cp src=/root/.cache/act/actions-upload-artifact@main/ dst=/var/run/act/actions/actions-upload-artifact@main/
[''] close /tmp/act834270224: file already closed
['']   🐳  docker exec cmd=[node /var/run/act/actions/actions-upload-artifact@main/dist/index.js] user= workdir=
['']   💬  ::debug::followSymbolicLinks 'true'
['']   💬  ::debug::implicitDescendants 'true'
['']   💬  ::debug::omitBrokenSymbolicLinks 'true'
['']   💬  ::debug::followSymbolicLinks 'true'
['']   💬  ::debug::implicitDescendants 'true'
['']   💬  ::debug::omitBrokenSymbolicLinks 'true'
['']   💬  ::debug::Search path '/tmp/artifact.tar'
['']   💬  ::debug::File:/tmp/artifact.tar was found using the provided searchPath
| With the provided path, there will be 1 file uploaded
['']   💬  ::debug::Root artifact directory is /tmp
| Starting artifact upload
| For more detailed logs during the artifact upload process, enable step-debugging: https://docs.github.com/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging#enabling-step-debug-logging
| Artifact name is valid!
['']   ❗  ::error::Unable to get ACTIONS_RUNTIME_TOKEN env variable
['']   ❌  Failure - Main Upload artifact
[''] exitcode '1': failure
['']   ❌  Failure - Main Upload artifact
[''] exitcode '1': failure
```

Searching online thankfully brought me directly to the issue, the most recent
[comment][gh_act_issue_ACTION_TOKEN_env] on which provides the solution: add
``--artifact-server-path /tmp/artifacts`` to the arguments for act

[gh_act_issue_ACTION_TOKEN_env]: https://github.com/nektos/act/issues/329#issuecomment-1187246629

It now builds successfully.

I briefly ran into issues with running it with sudo, those being the
permissions of files generated. When I tried to resolve the later deploy
issues, one of the first articles I found on the internet suggested that it was
an issue of running as root instead of the user. That wasn't it, I think. But I
did come to understand that running with sudo created the built files and
artifacts with root ownership. So don't do that if possible.

### Resolving deploy errors

I can't. That's the end of my effort to get this particular angle to work.

The runners correctly output all the content of the site to a tarball in
/tmp/artifacts, and extracting that and pointing a simple http server (such as
``python3 -m http.server``) at it works fine. The deploy step, however, wants
to create and use OIDC tokens for a cloud consumer. It's hard for me to
identify what exactly is going on with the deploy action, since it's all
written for node, and I believe is using some core libraries from node.js, and
I just don't know it that well. What I suspect is that the token which github
actions creates on its own systems for its internal API calls is capable of
being assigned the idToken: write permission.

Although creating a PAT worked to resolve token issues earlier, that was just
the existence of a correctly formatted token. This is, from everything I've
read, a different ball of wax.

In any event, the deploy build step really wants to be pushing everything to
github-pages as an external source. I don't want that, I want precisely not
that. I can't immediately identify what the correct alchemy to make it go
locally without alteration to the source code of the actions is, and I don't
*really* want to read through a bunch of javascript in order to make that work,
of which there is no guarantee anyway.

### Next steps

What I *did* find of great utility, however, is that the action to build jekyll
includes instructions to pull down and/or build a docker container which has
the correct dependencies that github-pages uses. *This* is the nugget I was
after anyway. Github's official advice of "build it with jekyll locally!" does
not necessarily include the full list of included and whitelisted dependencies
for jekyll which github-pages supports.

Ultimately the goal of this is learning to build a fungible development
environment. Since I have the source code to deal with building a jekyll
environment which has parity with the actions which build my github pages site,
I'm going to do just that. Probably jenkins is the next best step.
