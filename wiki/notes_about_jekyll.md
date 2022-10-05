---
---

# Jekyll Notes

## symlinks

Jekyll doesn't want to handle symbolic links for a number of reasons. Git
handles symlinks nicely, so it's unfortunate. The main cause for concern here
is that my diary entries are effectively my drafts for blog posts, and if I
want to edit one, I will want to edit the other. In order to deal with this, I
have them hardlinked on my local system. Probably what I should do is write a
makefile or some script which matches on checksum. I don't *have* to use the
vimwiki diary function, it's just convenient for me.
