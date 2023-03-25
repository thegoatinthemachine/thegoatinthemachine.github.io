---
title: AWK and Perl for record handling part 1
layout: post
excerpt_separator: <!--more-->
---

On my last contract I was hired to automate things with Powershell, which until
that point I had used sparingly, only in niche situations where I wanted a list
output of, e.g. Active Directory objects. During this time I grew to appreciate
the tooling around it, that cmdlets - the various tools written in powershell
or C# using the .NET/core system libraries - consistently behaved nicely with a
shared typesystem and outputs. I still have a love-hate relationship with
Powershell, but like Perl, it's a much nicer language than the previous shell
language.

<!--more-->

In the context of automation and scripting, the main value I found
was the simplicity, the delight in pulling some fields from some output and
munging them into a shape upon which I could act elsewhere. Casting between
types was frequently a crapshoot, but I appreciated being able to...

```powershell
$network_details_valid = [bool]$(
	Resolve-DnsName $current_record.name |
	Select-Object -ExpandProperty 'IPAddress' |
	Where-Object { $_ -eq $current_record.ip }
)
```

However, the dearth of Windows Powershell commands having been ported to pwsh 7
means, even if I had the inclination to install the language, runtime, and
associated risks, on my systems I would still need to be fooling around with
text output. (Re-)enter my first love Perl and its forebear AWK

## These languages?! What about something *modern*

AWK has frequently been maligned as an *AWKward* language, and Perl frequently
maligned as a write-only language. Without respect to their syntax, these are
peers as premier languages for record and text handling (or mangling) on un*x
systems. For AWK, this is its custom built purpose, to slurp up input
line-by-line, and on matches, perform some behavior. Perl, built partially as a
text-processing alternative to AWK and sed, borrows some of the ideas, like
BEGIN and END blocks. When I saw that Powershell had borrowed the same, I knew
what they were getting at. Dealing with streams and collections is what these
languages want to be doing.

Perl5 is no longer in vogue, I'm afraid, for a number of reasons, not the least
of which is the confusion around the Perl6 project, now Raku, between
2000-2019. I am honestly amazed at the popularity of Python, given that Python
went through similar development and compatibility hell between python2 and
python3, and is itself such an environment-sensitive language that (if rumors
are to be believed) Docker moved away from Python towards Go. Nonetheless, I
will basically always prefer languages which have more deliberate syntax like
Perl, or even Powershell, over the whitespace sensitive hell that is debugging
Python. But, okay, the pseudocode-is-actually-code language is pretty handy for
learning, so an entire generation of CS grads learned to love Python before
ever mangling memory with C.

In any case, in no part of this exploration am I going to claim any sort of
mastery over either AWK or Perl. As the adage goes, "there's more than one way
to do it", and someone who has been working with these languages exclusively
will likely better understand the idioms.

## AWKward

The focus of this article is *not* to teach about AWK. I have neither the
patience nor the expertise. The purpose here is to explore some techniques with
AWK beyond numbered field extraction. The gawk [manual][gnu-awk] is a very
well-written rundown of the language features, given is relative brevity. That
said... AWK's basic invocation from the command line is to pipe into the
command and provide it a program as string arguments with `/patterns/` and `{ statement blocks }`.

Inconveniently, bash and AWK use the same leading sigil, `$`, so you'll
single-quotes so that the AWK command string doesn't get interpolated by the
shell environment, and double-quotes if you want this interpolation, e.g, to
use an environment variable with more pleasant syntax. This problem can crop up
with perl and powershell, also, given they also use sigils in this way, but
this is primarily a problem when using a one-line program at the prompt. The
particulars of this change depending on the implementation. In my case, I
happen to be composing this post from macOS, so it's using Apple's supplied
AWK, which is evidently based on *The one true AWK*. [Apple's developer page][apple-awk]
recommends reading the manual, and reading the [gawk manual][gnu-awk], even
though gawk is technically an extension in many ways from POSIX AWK. The gawk
manual has some excellent pointers about dealing with this at the prompt.

[apple-awk]: https://developer.apple.com/library/archive/documentation/OpenSource/Conceptual/ShellScripting/Howawk-ward/Howawk-ward.html
[gnu-awk]: https://www.gnu.org/software/gawk/manual/html_node/index.html

The following example is typical usage of AWK, pipe some tabular output to it,
filter on some rule, print some numbered fields.

```bash
# retrieve pid of any process which the current user (default) is running whose
# invocation line matches 'bash'

match_string='bash'

ps | awk "/$match_string/" '{print $1}'
#        ^ bash var interpolation  ^ awk field index var
```

## Using records to our advantage

Note that AWK processes a *record* at a time, delimited by the record
separator, and shoves the whole record into the 0th index, `$0`, and each
field, as delimited by the field separator of that record into all following
indices.

This works quite well on nicely behaved processes that produce columnar output,
as most unix tools are wont to do. Some, however, provide records in some
verbose list rather than tabular format. For example, `VBoxManage list hdds`:

```
UUID:           1eafcbe0-6e3f-43c5-968d-b7716bbcea54
...
Location:       /Users/goat/VirtualBox VMs/lab-server/lab-server.vdi

UUID:           235daf77-c6f5-4d8b-b916-2818628af9a0
...
Location:       /Users/goat/VirtualBox VMs/slack/slack.vdi
```

### handling this example with pipes

There are of course, numerous ways to manipulate this text. The simplest way
for my brain to handle, if I'm looking for the UUIDs of all records where the
location matches the string 'slack', would be to lay some piping:

```bash
UUIDs=( $(
	VBoxManage list hdds |
	grep -B5 -e 'slack' |
	awk '/^UUID/ {print $2}'
) );

echo "${UUIDs[@]}"
```

`grep` is also providing in its output the 5 lines of context *Before* the
match, and `awk` is regex-matching against any lines that start with 'UUID',
and printing the 2nd field. There's most likely some ugly multi-line regex with
lookahead which can accomplish this more cleanly, but this will do fine.

This still runs into the same problem of me, a stupid human, trying to keep
track of a number of lines and fields. It's not as precise as targeting a field
by name, or perhaps, offset.

### straight AWK multiline records

The gawk manual has good coverage of [how multiline records][gnu-awk_multiline]
can be achieved. The basic gist is setting the *Record Separator* (RS) before
program execution in some way to be the empty string, `""`. I choose to use a
BEGIN block because I perceive it as less noisy than multiple `-v` flags, but
at this level of complexity, which is mild in the scheme of things, it all
starts to look a little noisy.

[gnu-awk_multiline]: https://www.gnu.org/software/gawk/manual/html_node/Multiple-Line.html

```bash
awk 'BEGIN {RS=""; FS=":[ \t]+|\n"; OFS="="} { for ( i = 1; i <= 10; i += 2 ) print NR" "$i,$(i+1) }' vbox_hdds.txt

# produces:
# 1 UUID=40311a77-d05b-4787-b8fe-66df889598f0
# 1 Parent UUID=base
# 1 State=created
# 1 Type=normal (base)
# 1 Location=/Users/goat/VirtualBox VMs/slack/slack_1.vdi
# 2 UUID=1eafcbe0-6e3f-43c5-968d-b7716bbcea54
# 2 Parent UUID=base
# 2 State=created
# 2 Type=normal (base)
# 2 Location=/Users/goat/VirtualBox VMs/lab-server/lab-server.vdi
# 3 UUID=235daf77-c6f5-4d8b-b916-2818628af9a0
# 3 Parent UUID=base
# 3 State=created
# 3 Type=normal (base)
# 3 Location=/Users/goat/VirtualBox VMs/slack/slack.vdi
```

NR here is the "Ordinal Number of the Record", i.e, to which record the field belongs.

I'm pulling some tricks to make this work more smoothly in this demonstration,
because I didn't want to demonstrate with every field on its own line.

Because the default *Field Separator* (FS) is whitespace, the fields would
normally be split very aggressively, in this case, chopping some fields in
half, like "Parent UUID:" and any locations which have whitespace in the path,
such as the "VirtualBox VMs" directory. If I set the FS to be ":", this would
prevent splitting those fields with interior whitespace, but this also keeps
*all* the whitespace which might otherwise be omitted from a field. Initially I
went down the path of attempting to trim the text of the fields, but that was
unproductive and clunky. I'm resolving this by setting the FS with a regex:
`":[ \t]+|\n"`. The OR operator `|` here allows for a field to be separated by
*either* a colon followed by at least one space or tab *or* by a newline.

I happen to know that I can print out the results I want, in this case, by
targeting the 2nd and 10th field of each record:

```bash
awk 'BEGIN {RS=""; FS=":[ \t]+|\n"} $0 ~ /slack/ { print $2,$10 }' vbox_hdds.txt
```

But this still requires more precision than I want to have to express. I'm a
stupid human, I don't want to be *counting*, that's what the computer is for.
What I want is to snag something on property name. Conveniently, AWK arrays are
really hashes, able to be indexed by numbers and by strings equally well. A
similar walk through the fields can assign an array with keys matching the
properties, and I can then pull the information for the fields I want. It's a
naive approach, but it works.

```bash
 awk 'BEGIN {RS=""; FS=":[ \t]+|\n"} $0 ~ /slack/ { for(i=1;i<=NF;i+=2) keys[$i]=$(i+1); print keys["UUID"], keys["Location"] }' vbox_hdds.txt
 
# produces:
#
# 40311a77-d05b-4787-b8fe-66df889598f0 /Users/goat/VirtualBox VMs/slack/slack_1.vdi
# 235daf77-c6f5-4d8b-b916-2818628af9a0 /Users/goat/VirtualBox VMs/slack/slack.vdi
```

However, this is starting to bleed into more than one-liner territory, wanting
of a here document if one wanted to embed this in a bash script. This
implementation here, using a short filename as input instead of the pipe,
still totals to 142 characters. Ain't nobody got time for that on an 80 column
terminal. In any event, this is basically like any operation to deserialize
data.

Once the desired data is extracted, one may wish to use it in some structured,
named fashion. Insofar as a shell language like bash goes, this way lies
madness. Bash didn't support hashes until v4, which does not come standard on
macOS, still shipping some v3. Although I have bash 5 installed, even so,
multi-dimensional arrays are not supported in some clear syntax, instead
relying on a workaround I learned back the first time I took a unix class,
which is to create a subscript with a comma between the keys, such as
``declare -A mdarray='(["$i,$k"]=$foo)'``. Bash is not my language of choice
for dealing with complex structures in my data-munging operations.

However, if for some insane reason, one wishes to export this structured data
to bash, it requires some goofy quote escape logic:

```bash
declare -A hdds="($(awk -v q="'" 'BEGIN {RS="";FS=":[ \t]+|\n";OFS=" ";} {for(i=1;i<=NF;i+=2) printf(" [\"%d,%s\"]=\"%s\" ",NR,$i,$(i+1))}' vbox_hdds.txt))"

# declare -A hdds="($( # wrap array operator `(' in double-quotes
# awk ' # begin single-quoted commands
# BEGIN {RS="";FS=":[ \t]+|\n";}
# {for(i=1;i<=NF;i+=2) printf(" [\"%d,%s\"]=\"%s\" ",NR,$i,$(i+1))}
#                               ^ ["keya,keyb"]="value"
# ' vbox_hdds.txt
# ))"
```

I do not recommend this, as it is hell to debug.

If you want to verify all that crap works, this should produce an (unsorted)
printout of this associative array:

```bash
for entry in "${!hdds[@]}"; do echo $entry: "${hdds[$entry]}"; done
```

That is, in all, a shocking amount of line-noise in order to deserialize data
into something resembling a structured format. Next, I'll explore Perl's
capability, because the number one thing that makes Perl stand out above most
other language for text processing is its quotation operators, which make all
that crap significantly less noisy.
