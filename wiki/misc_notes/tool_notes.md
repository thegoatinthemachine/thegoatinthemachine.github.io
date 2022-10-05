---
---
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

*before* I call ``colorscheme gruvbox``.

In the event that undercurls are desirable, the vimhelp document about the
:hilight section notes these commands:

```vim
let &t_Cs = "\e[4:3m"
let &t_Ce = "\e[4:0m"
```

[gh-gruvbox-spellcheck-colors]: https://github.com/morhetz/gruvbox/issues/372#issuecomment-743232530

### grep

#### colors and pagers

``grep ... --color`` can be used to make the output more readable in the
terminal, it defaults to 'auto'. It also accepts 'always' and 'never':
``--color=always``

If you want to pipe that to a pager like less, however, the 'auto' default will
turn color off, since it uses terminal control codes to effect colors. This is
a sensible default, as most programs will not recognize color control codes.
Less can deal with colors if you tell it to interpret raw control codes,
however, if you use ``-R``:

```bash
grep ... --color=always | less -R
```

less will render this piped output nicely, which depending on your version of
grep can include highlighting the search string, the filenames, etc.
