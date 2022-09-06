---
title: Can into CI and local testing
layout: post
excerpt_separator: <!--more-->
---

## More about deployment errors

AFAICT, the action for deploy-github-pages specifically wants to interact with
the github API in order to push to github pages. I specifically don't want
that, because I want to develop and test locally (or at least in a fashion
which can be adapted besides github's environment). If I understand correctly,
there may be a way to configure that action to deploy to other providers, like
Azure, but the documentation is sparse.
<!--more-->
 
I *could* attempt to maintain multiple versions of the github provided action
file, pages.yml, and merge selectively from a dev branch to pages with ``git
checkout --patch``. I don't want to do that either, really. The whole shebang
just remains far too tied to the github-actions workflow if I do.  Moreover,
this whole process is still in beta at the time of writing. Better to learn
from what actions are being taken, outlined in the previous entry, and adapt to
a more mature system over which I have more direct control, and which is less
fragile to my locally screwing around.

Local or self-hosted [runners][gh_docs_self_hosted_runners] are maybe an
option. For the scope of this project, getting that working seems to me like
wasting time. Learning it will hopefully be involved for my career down the
line, but right this moment, that's *way* out of scope.

[gh_docs_self_hosted_runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners)

## The right image with the right dependencies

Reading through the process, I previously found that github maintains a
container which has all the jekyll dependencies and whitelisted plugins already
built. This is hosted on githubs own package system, ghcr.io, as
https://ghcr.io/actions/jekyll-build-pages the repository for which can be
found [here][gh_pkgs_jekyll_build].

This is, per their documentation, just straight up a docker image which can be
pulled by URL, in which commands can be run. Actually building the site with
jekyll is the important bit, just about anything can serve it locally.

[gh_pkgs_jekyll_build](https://github.com/actions/jekyll-build-pages/pkgs/container/jekyll-build-pages)

## Serving the files

While this doesn't tell me much about the hosting environment, most such
hosting environments are opaque: upload some static site files, with html, css,
and optionally client-side javascript, and let the site serve pages. That's why
they offer such things for free, sites are by definition not getting any
backend processing besides serving. S3 buckets, DigitalOcean droplets, gitlab,
and github pages just take in the site files.

I'll worry about how to actually serve the files later, for now the previously
mentioned ``python3 -m http.server`` will be fine.
