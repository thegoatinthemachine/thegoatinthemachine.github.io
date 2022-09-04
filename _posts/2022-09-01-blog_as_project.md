---
layout: post
title: "This blog is another project lol"
---

Blog -> weblog, in this case, build log

I will soon add this site as an entry to my homelab projects document. My plan
is to do writeups of each of those projects in some amount of detail, and
though I may or may not do those in their own posts or as part of the blogging
function of jekyll, since this website is an active project, it makes perfect
sense to blog as I go.

So what's the first project I want to deal with on this site?

Make it pretty? Well, eventually, yes, but first...

Problem: I want to see and verify changes before I push them to the live site,
which at time of writing is hosted with github-pages, which is using the
standard jekyll structure.

Solution: render and serve in jekyll locally, duh.

Complication: I want to use a development environment that's clean and
functionally identical to github-pages.

Because I see an opportunity to learn, I want to pull the classic SysEng
gambit: why bother doing something in 5 minutes when I can spend 5 hours
failing to automate it? Half the point of this site is as an active blog and CV
for my IT career. The other half is learning. If you checkout this version repo
and build it, you'll see it as barebones. I deliberately left a bunch of stuff
up to the defaults, so I could understand how jekyll works.

Enter ``act``. [act][gh/nektos/act] purports to be capable of running github actions locally.

[gh/nektos/act](https://github.com/nektos/act)

``act`` has a dependency on Docker, since it uses the same containerization
idea that github-actions do. That's a good place to start.
