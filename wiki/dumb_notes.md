---
---

## distro notes

### Manjaro

#### Installing a package from AUR (Arch User Repository)

TL;DR: Use pamac

The AUR is outside of pacman's scope, as it is not really a "repository" in the
same way that the arch or manjaro repos are. It instead contains individual git
repos and PKGBUILDS, which describe to pacman how and where to grab (and/or
build) binary packages. the [archwiki details][aw-AUR_helpers] several
AUR helper programs, which either wrap around pacman and provide searching of
the AUR, or similarly wrangle grabbing the git details from AUR and shoving
them into pacman.

pamac is one such tool built by the Manjaro team. It has a GUI program and a
CLI program. These can be configured to use the AUR. It also supports flatpack,
snap, etc, and has multithreading downloads built in.

[aw-AUR_helpers](https://wiki.archlinux.org/title/AUR_helpers)

## tool notes

### vim

#### gruvbox spellcheck weirdness in tmux over ssh and etc

TL;DR: Set the hilight colors for bad spelling in the vimrc

Solution can be found [here][gh-gruvbox-spellcheck-colors]. Basically, gruvbox
is using some fancy features, particularly undercurls, which may not be
supported in the terminal. This can be worked around by setting the SpellBad
colors after the colorscheme is called.

```vim
hi SpellBad cterm=underline ctermfg=red
```

In my testing my vimrc, I found it important to call

```vim
filetype plugin on
syntax enable
```

*before* I call ``colorscheme gruvbox``. In the event that undercurls are
desirable, the vimhelp document about the :hilight section notes these
commands:

```vim
let &t_Cs = "\e[4:3m"
let &t_Ce = "\e[4:0m"
```

[gh-gruvbox-spellcheck-colors](https://github.com/morhetz/gruvbox/issues/372#issuecomment-743232530)
