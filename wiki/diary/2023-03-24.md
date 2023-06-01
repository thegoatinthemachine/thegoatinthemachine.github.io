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
means that even if I had the inclination to install the language/runtime, and
accept associated risks on my systems, I would still need to be fooling around
with text output out of my standard unix tools. (Re-)enter my first love, Perl,
and its forebear AWK

## These languages?! What about something *modern*

AWK has frequently been maligned as an *AWKward* language, and Perl frequently
maligned as a write-only language. Without respect to their syntax, these are
peers as premier languages for record and text handling (or mangling) on *nix
systems. For AWK, this is its custom built purpose, to slurp up input
line-by-line, and on matches, perform some behavior. Perl, built partially as a
text-processing alternative to AWK and sed, borrows some of the ideas, like
BEGIN and END blocks. When I saw that Powershell had borrowed the same, I knew
what they were getting at. Dealing with streams and collections is what these
languages want to be doing.

Perl5 was once king, but is no longer in vogue, I'm afraid, for a number of
reasons, not the least of which is the confusion around the Perl6 project, now
Raku, between 2000-2019. I am honestly amazed at the popularity of Python,
given that Python went through similar development and compatibility hell
between python2 and python3, and is itself such an environment-sensitive
language that (if rumors are to be believed) Docker moved away from Python for
its implementation towards Go. Nonetheless, I will basically always prefer
languages which have more deliberate syntax like Perl, or even Powershell, over
the whitespace sensitive hell that is debugging Python. But the
pseudocode-is-actually-code language is pretty handy for learning, so an entire
generation of CS grads learned to love Python before ever mangling memory with
C.

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
command and provide it a program as string arguments with conditions including
`/patterns/`, and correlated expressions as `{ statement blocks }`. Omitting a
condition and defining a statement effectively allows it to always execute, and
conditions need not be regular expressions, and can reference the record and
field variables set automatically for each new record in the condition's
evaluation, such as `NR == 1`

Inconveniently, bash and AWK use the same leading sigil, `$`, so for one-liners
you'll need single-quotes so that the AWK command string doesn't get
interpolated by the shell environment, and double-quotes if you do want this
interpolation, e.g, to use an environment variable with more pleasant syntax.
This problem can crop up with perl and powershell, also, given they also use
sigils in this way, but this is primarily a problem when programming one-liners
at the prompt.

The particulars of these sorts of AWK problems change depending
on the implementation. In my case, I happen to be composing this post from
macOS, so it's using Apple's supplied AWK, which is evidently based on *The one
true AWK*. [Apple's developer page][apple-awk] recommends reading the manual,
and reading the [gawk manual][gnu-awk], even though gawk is technically an
extension in many ways from POSIX AWK. The gawk manual has some excellent
pointers about dealing with this at the prompt.

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

This works quite well on nicely behaved programs that produce columnar output,
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

This sort of output can be handled with relative ease in oneliners by adopting
some techniques for the tools.

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
track of a number of lines and fields. This requires, strictly speaking, more
precision than I want to express.

### straight AWK multiline records

The first issue is `grep -B5`. I don't want to be thinking about how many lines
composes a record, especially not if the level of verbosity changes the output.
Columnar output is all one-record-per-line, but AWK can handle multiline, or
paragraph, records just fine as long as you know the delimiter.

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

I'm pulling some tricks to make this work more smoothly in this demonstration,
because I didn't want to demonstrate with every field on its own line.  NR here
is the "Ordinal Number of the Record", i.e, to which record the field belongs.
OFS also influences the print statement as "Output Field Separator"; the print
statement separates comma-separated arguments with this value.

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

### getting fields

This still requires more precision than I want to have to express. I'm a
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
implementation here, using a short filename as input instead of the pipe, still
totals to 142 characters. Ain't nobody got time for that on an 80 column
terminal. With awk, one should also be able to just leave the expression
argument unclosed across multiple lines.

## dealing with columnar output

Columnar output with a header row is significantly easier to wrangle by name,
in my opinion. For one, it doesn't require any goofy increment math, you're
just taking the first row's fields as keys for a hash, and storing the index at
which they appear, and from there specific fields can be pulled by name.

`ps` is a good example, very nicely behaved with a header row. We can target
just that row so we know the keys.

```bash
ps -v | awk 'NR==1 {print}'
```

We can take the naive approach for a small list of keys, which is pretty brief,
only falling out of an 80 column terminal by a little bit.

```bash
ps -v | awk 'NR == 1 {for (i=1; i<=NF; i++) keys[$i]=i} {print $keys["TIME"], $keys["PID"]}'
```

But if you are like me, and you're scrolling up through history and not wanting
to do a large amount of line-editing each time you have a new oneliner, you can
do this sort of thing:

```bash
ps -v | awk -v cols="TIME,PID,%CPU" 'BEGIN {split(cols,props,",")} NR == 1 {for (i=1; i<=NF; i++) keys[$i]=i} {for (k=1; k<=props; k++) printf("%8s ", $keys[props[k]]); print ""}'
```

This will define upfront, in the `-v` argument cols, the order in which named
columns are desired. This has the downside of not checking for existence, and
the downside of being obnoxiously long, wrapping around an 80 column almost
twice. Now, we can put newlines between statements in a string being fed to
awk, even on a REPL prompt, but that's also fraught. I prefer in such
circumstances to drop into an editing session with "edit-and-execute-command"
in bash.

Note the C-style for loop in these. Awk also knows the shell-ish "for (elem in
array)", but because the arrays here are actually hashes, their subscripts
being strings, there is not necessarily a guarantee that they'll spit out in
the order that you insert them. So that unpleasantly fiddly C-style for loop
remains in all locations where I might hope to, in a different language, say
"for (i in 1..len(array))" to walk an integer in order through the indices, or
something of the sort. Awk does not have any such brace-expansion functionality
as bash does.

## structuring the output data in bash

Once the desired data is extracted, one may wish to use it in some structured,
named fashion. Insofar as a shell language like bash goes, this way lies
madness. Bash didn't support hashes until v4, which does not come standard on
macOS, still shipping some v3. Although I have bash 5 installed, even so,
multi-dimensional arrays are not supported in some clear syntax, instead
relying on a workaround I learned back the first time I took a unix class,
which is to create a subscript-string with a comma between the keys, such as
``declare -A mdarray='(["$i,$k"]=$foo)'``. Bash is not my language of choice
for dealing with complex structures in my data-munging operations.

A note about shells: Zsh is now the standard shell that ships with macOS, which
is because of GPL reasons if I understand correctly, bash v3 being the latest
revision not to have GPL licensing of the sort that makes apple unwilling to
play ball. Zsh's designers thought it wise to use the same Associative-Array
declaration syntax as bash, so `declare -A hash=["keya,keyb"]` works equally
well to achieve this pseudo multi-dimensional array effect. I'd still prefer it
if they allowed explicit subscripting such as `array[indexa][keyb]` as you find
in C-like languages, but some of that gets hinky depending on backwards
compatibility and the underlying math of how arrays are allocated. In any case,
true multi-dimensional arrays are not possible, nor are array references, so
we're stuck emulating a multi-dimensional array.

If for some reason, one wishes to export this structured data to bash the
shell, it requires some goofy quote escape logic:

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

Part of what's going on here is the magic with that `printf`: `printf` doesn't
automatically tack a newline on the end of a print operation. Because that
whole awk command is wrapped in command substitution `$()`, and that wrapped in
the array or list operator `()`, bash treats this huge string of
`[n,key]=value` entries as a valid array assignment.

If you want to verify all that crap works, this should produce an (unsorted)
printout of this associative array:

```bash
for entry in "${!hdds[@]}"; do echo "$entry: ${hdds[$entry]}"; done
```

## array walking a fake multi-dimensional array

More should be said about tracking down the specific keys for a specific
record, which is frequently useful. This gets real hairy and unpleasant real
fast. These pseudo md-arrays are being constructed with Associative Arrays,
bash's language for a hash or a dictionary. That means that
retrieving/manipulating string-based keys for them is a lot of work, dealing
with bash's limited string handling capacity. This is entirely ironic, IMO,
because the next language I'd be using - Perl - has incredibly good support for
string handling, but makes all of this crap entirely unnecessary, since its
syntax explicitly supports lists of lists. (The backend of how Perl's data
structures of this type works are a little squirrelly, but I'll get to that).

Anyway, bash had wildcard-glob matching - e.g, `[[ $string == *.sh ]]` - since
functionally forever (I have no idea of if this is an OG bourne shell feature),
and some limited regex e.g, - `[[ string =~ ^$ ]]` - capacity since at least
some early version of bash v3. We're pretty likely to find the capacity in any
bash environment of the last 20 years, anyway. Either way, we can build a
conditional when enumerating the entries in a list, and pull just the ones
which match a glob or regex pattern:

```bash
record='1'
for entry in "${!hdds[@]}"; do
if [[ $entry =~ $record,.+ ]]
then echo "$entry: ${hdds[$entry]}";
fi;
done
```

Alas, because this is a fakery, we can't get the keys of the individual
records, as you might hope for with something like `keys(hdds[0])`. To get the
keys we have to do some ugly bash string manipulation.

```bash
keys=(); for entry in "${!hdds[@]}"; do IFS="" keys+=("${entry/#*[1-9],/}"); done; unset IFS
```

It's so gross. IFS needs to be set to something other than ' ' whitespace so
that keys with whitespace process properly; no other combination of quotes
seemed to have the magic. I dislike that I have to do this, but bash is
whitespace sensitive, so here we are. That `"${entry/#*[1-9],/}"` bit is some
bash string expansion while doing a replacement, and if I could get it to pass
each argument properly, including those broken up by whitespace, I would have
done that. The bash manual has more information on the subject, but effectively
`${parameter/#pattern/string}` will match a pattern at the beginning - via
'/#', '//' or '/%' have different effects - of the value of parameter and
replace that with 'string'. If string is null or omitted, it deletes it.

There are then several ways by which this might be reduced to a unique list.

The easiest I found was to pipe from printf to sort:

```bash
unique_keys=$( printf '%s\n' "${keys[@]}" | sort -u )
# This will not work if any of your keys contain newlines
```

With that we can use the keys and our ordinal record number to compose our
string keys for indexing the fake multi-dimensional array.

It's important to note in this that these operations are all possible by other
means. It's entirely possible to build complex data structures in bash by
using, e.g, multiple sets of related arrays, but that relationship being built
is entirely a consequence of the programming techniques to handle them well,
nothing semantically meaningful to the language environment. What I'm trying to
accomplish here is to retrofit onto the shell some oneliner-capable data
handling, which necessitates brevity.

## part 1 conclusion

That is, in all, a shocking amount of line-noise in order to deserialize data
into something resembling a structured format. Next, I'll explore Perl's
capability, since I believe perl is the single language best-suited to this
sort of fiddle-farting around in a text-oriented shell. Its quote-like
operators are native to the language environment, and make many of problems of
ugliness go away. Ruby is smart enough to copy some of that, but not enough to
supplant perl for me.
