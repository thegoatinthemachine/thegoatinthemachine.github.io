---
title: Breaking down what I need to replicate
layout: post
excerpt_separator: <!-- more -->
---

The behavior of import to me at the moment, concerning configuring jekyll and
the jekyll build environment, are two github actions. They're described in the
file .github/workflows/pages.yml as such:

```yaml
      - name: Setup Pages
        uses: actions/configure-pages@v2
      - name: Build with Jekyll
        uses: actions/jekyll-build-pages@v1
        with:
          source: ./
          destination: ./_site
```

<!-- more -->

``act``, although it couldn't get me the whole way, was capable of pulling down
and caching those actions from the actions repositories. AFAICT, the assumed
prefix is just github.com E.g,
[actions/configure-pages][gh_action_configure_pages]

In the last entry, I was making some assumptions based on the material I was
seeing in those respective action.yml files.

```yaml
runs:
  using: 'node16'
  main: 'dist/index.js'
```

So, I looked at dist/index.js file. It is unholy. The callgraph for it is
unreasonably large and complex and I'm convinced was dreamed up by some
abominable thing. I saw that and tried to read it and realized I was too tired
to approach the task. Then I took another look at it today and realized it was
produced by webpack, which incorporates all the modules necessary,
including the github actions core module. So I wasn't too far off :P

[gh_action_configure_pages]: https://github.com/actions/configure-pages

Anyway. There's also a src/index.js, and associated files. That's much more
sane.

## examining the code

I mentioned the callgraph for the webpacked dist/index.js file because I wanted
to understand what was going on in that file. I'm not super familiar with a
large variety of javascript tooling, but I found an extremely useful script
which did precisely what I need, and which I'll definitely be using going
forward.

### [``callGraph``][gh_koknat_callGraph]

It's a perl script. The readme indicates that it goes through line-by-line with
a regex scraping for definitions and function calls, building a graph out of
them for the .dot graphviz suite.

I'm not a fan of the .dot files that this tool generates, because they're full
of very specific spacing, location, and coloring directives. That dampens the
ability to use it with a wider suite of tools. But, I've written plenty of
graphs, including ERDs in .dot files.

It has dependencies on the graphviz suite, and the graphviz perl interface,
which is on CPAN as 'GraphViz'.

It's been a hot minute since I've installed anything from CPAN, and I've never
configured it for my laptop. I didn't want to need to configure it, let alone
reading about how to configure it for the whole system rather than an
individual user. ``cpanminus``, thankfully, cut that work short, and someone
else already wrote a homebrew formula for cpanminus. So my install process was
as such:

```bash
# satisfy dependencies
brew install graphviz cpanminus
sudo cpanm GraphViz # to install system-wide
# whenever I pull tools from github, I try to just clone them where appropriate
# rather than muck around with my $PATH, since I already use /usr/local/bin for
# homebrew, I tend to symlink tools. stow is convenient for that
cd /usr/local/stow/github && git clone https://github.com/koknat/callGraph
stow -d /usr/local/stow/github/koknat \
	-S callGraph \
	--ignore="t|.png" \
	-t /usr/local/bin/ ;
```

This is condensed, in reality this was all from the prompt doing dry runs with
verbosity before going ahead.

[gh_koknat_callGraph]: https://github.com/koknat/callGraph
