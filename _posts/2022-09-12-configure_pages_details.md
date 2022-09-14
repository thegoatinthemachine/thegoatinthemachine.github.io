---
layout: post
title: Details of pages and jekyll build workflow
excerpt_separator: <!-- more -->
---
# {{ page.title }}

So the callGraph script is neat, but nobody can be expected to have in-depth
knowledge of every language they're attempting to address at first blush. The
initial graphs this produced for me were limited, and didn't list all of the
function calls. I found that the regular expressions callGraph uses to
recognize function definitions can be extended with optional non-reference
generating atoms. In this case, it was missing an understanding of javascript
async functions, so I did a quick fork and pull request to insert

```
...(?:async\s+)?...
```

<!-- more -->

After this addition on my local fork, callGraph now successfully recognizes
async function definitions, of which there are many in this github actions
workflow. callGraph tries to match up specific function definitions and
function calls, and without being able to recognize both a definition and a
call, it will fail to illustrate that connection. I'm certainly not going on
some quest to extend it for all of the myriad ways in which javascript allows
functions to be defined or called, that would be the way of madness, and far
outside the scope of this project. I just need it 'good enough'

Having extended it, I now have a more complete call graph:

![github actions 'configure pages' callgraph][gh_ac_config_pages_2]

[gh_ac_config_pages_2]: /assets/images/2022-09-12/gh_act_config_pages_2.png

With the ``-verbose`` flag, callGraph also lists the externally referenced
files, such as modules. It assumes, however, that such files will end in a file
extension matching the passed ``-language`` argument. So, it sees and tells me
about the references to the various nuxt.js and etc files, but because those
are merely read from in order to populate a blank configuration file, and have
no functions associated with them, they don't show up on the call graph. I know
to look for their references, though.

## Configure pages meat and potatoes

There are several operations about which I do not care. I got really lost in
the weeds trying to figure that out. The configure-pages action has several
outputs, and as far as I can tell, none of the other actions consume them, per
their respective ``action.yml`` files. The main thing that configure-pages
seems to be concerned with for the purposes of the workflow, is calling out to
the github api and making sure that there exists an endpoint to which the built
site can be pushed and hosted, enabling the functionality if needed by using a
github token, either one auto-generated in the context of the action runner
which it would be if it's hosted by github, or a personal access token
otherwise. Essentially, none of the action for configuring pages is relevant to
local development, it's almost all external to the cloud provider, in this case
github pages hosting per their [API](api.github.com), or informative.
Otherwise, it's establishing basic config files for nuxt, next, sveltekit, or
gatsby, none of which are relevant here.

There are several things I learned while I was up to my eyeballs in it,
however.

## What I learned about Github Actions

So an ``action.yml`` file defines a bunch of properties which are relevant to
the actions runner. Locally, this is still my installed version of
[nektos/act](https://github.com/nektos/act). Critically, this file defines
inputs, if they are required, and their defaults. I passed ``act`` the debug
flags and captured all output to a log, and this debugging output is fairly
thorough, so I'm seeing that there are no inputs of the kinds I think are
missing being passed.

As I was digging into the configure-pages action, I found that one of the first
``index.js`` calls is to ``getContext()``, which itself calls
``getRequiredVars()``. ``getContext()`` might better be understood as
validateContext(), since that's its primary function. If one of the required
variables is undefined, it throws an error and the whole operation halts.

But

There's not a way for that to happen under normal circumstances. I certainly
saw that, and thought, "now wait a second, ``getRequiredVars()`` checks through
several inputs I haven't supplied, including ``static_site_generator``, which
is only used if the generator is *not* jekyll. No default value is specified,
why *isn't* this failing?"

### Glory of open source

My answer was actually in the core github
actions toolkit. I'll explain, but I missed the answer the first readthrough.

My first thought was that in ``action.yml``, the actions runner is configured
to use the boolean parameter ``required`` if a ``default`` is not set. So, I
searched for "input default when required false", and found several results.

Critically, a stackoverflow [question][so_gh_action_required_inputs], github
[issues][gh_iss_action_validate_inputs], and some github discussion, pertaining
to validating that required input is actually passed in.

[so_gh_action_required_inputs]: https://stackoverflow.com/q/68804484 
[gh_iss_action_validate_inputs]: https://github.com/actions/runner/issues/1070

This only gave me partial insight. For one, that issue, I think, was moved from
the [actions/toolkit/core][gh_actions_toolkit_core] repository to the
actions/runner repo, where it now lives. Per the context clues of the
discussion, their saying that it's out of scope for the toolkit.

In any event, they pointed to the ``getInput()``
[function][gh_actions_core_getInput] from the core repository. If you are
paying attention, you'll see the answer right away.

[gh_actions_toolkit_core]: https://github.com/actions/toolkit/tree/main/packages/core
[gh_actions_core_getInput]: https://github.com/actions/toolkit/blob/e6257f111756d2f3567917c8e27ab57de8c3e09c/packages/core/src/core.ts#L134-L155

### I didn't see the answer right away

Surely, I thought, since this is a runner issue, as they say, I should check
the runner source code. The runner is evidently written in C#. Although I've
not written C#, honestly, imperative-OOP language A is imperative-OOP language
A. This is not crammed with weird language idioms, it's fairly straightforward.

Through some hunting around with github's search, I found the function
[call][gh_action_runner_evaluate_input_default] that actually processes the
default value for an input. As my hypothesis as yet unchanged goes, if the
actions runner is silently inserting the value of the ``required`` parameter
when a default is not found, this is where it will be.

It's not, but it does refer to a unique token I can search for,
``input-default-context``. That leads to a json schema
[definition][gh_action_runner_action.yml_schema].

[gh_action_runner_evaluate_input_default]: https://github.com/actions/runner/blob/f9c2bf1dd72541bf039c3c5fa4129814181ca261/src/Runner.Worker/ActionManifestManager.cs#L276-L302
[gh_action_runner_action.yml_schema]: https://github.com/actions/runner/blob/ead3509d5a37090dac954dd7aae6dcba468b5915/src/Runner.Worker/action_yaml.json#L23-L31

Damn. The action runner pulls the input default from the ``default`` parameter,
and nowhere else. Well, that's the github action runner source code, what
about ``act`` source code?

### The answer wasn't in act source either

C# is pretty conventional if you know OOP and imperative code. It's
fundamentally similar to java, for instance. Go is completely new to me, so it
took some amount of reading before I was really grasping what was going on. The
callGraph for ``act`` is larger than I feel comfortable putting on a freely
hosted site. It's huge.

Between reading through the callGraph and some targeted searching, I'm
reasonably confident that I understand how ``act`` processes inputs from
action.yml as well. My understanding is as follows:

1. In the runner package,
   ``[setupActionInputs()][gh_act_runner_action_inputs]`` gets a function for
   how to step through an action.yml file from the appropriate model source
2. and fills an object ``action`` with the contents of the actions file
   accordingly, including several input keys by their ids.
3. It then interpolates the default value for that input, if it exists, or the
   input if applicable.

Walking through the [model][gh_act_model_action], it also defines the default
input based on the associated ``default`` parameter in ``action.yml``

[gh_act_runner_action_inputs]: https://github.com/nektos/act/blob/3a0fe6967fd8ecd3cb86550d0225d55a0bb37ac9/pkg/runner/action.go#L375-L396
[gh_act_model_action]: https://github.com/nektos/act/blob/943a0e6eea2f67783018b9d3bc375a6a7dd65ab3/pkg/model/action.go

### The answer was in the first place I looked

At this point I've got some understanding that, in fact, neither ``act``, nor
github actions does any kind of sneaky backfilling of the defaults. They both
read *only* from the default value for the input. ``act`` may or may not
actually respect YAML formatted variables, since I only saw it interpolating
from the default parameter, but that's more than likely me just not
understanding go that well. So what's going on? Why, when I'm not giving any
input, nor is the action.yml defining a default, is ``getContext()`` not
throwing errors at undefined variables?

Back in github actions [core][gh_actions_core_getInput], my eyes skipped right
over what's going on. I correctly took in information about checking if the
input has the ``required`` parameter true, and if it has no input then throwing
an error. What I missed was the ``||`` condition. If there is no environment
variable matching the one named, such as INPUT_STATIC_SITE_GENERATOR, *getInput
populates the local variable with an empty string*!

An empty string, you will notice, is *not* undefined. D'oh.

However, an empty string *does* count as false-y for javascript, which is why
when index.js/main() checks ``if (staticSiteGenerator)`` in a single
conditional later, it fails and moves past that block. js is an abomination,
and there's just so much implicit behavior going on here, driven by language
idioms.
