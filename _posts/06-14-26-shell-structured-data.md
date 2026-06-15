---
layout: post
title: Quoted Data Streams for Shell Scripts
date: 2026-06-14
---

  Take a look at this shell snippet:
```bash
for f in $(ls); do
  if [ -d "$f" ]; then
    echo "Directory: $f"
  elif [ -f "$f" ]; then
    echo "Regular file: $f"
  else
    echo "Not recognized: $f"
  fi
done
```

  Seems pretty harmless on the surface, but you may have noticed the papercut you may receive if you run this:

```bash
$ ls
some_dir
some_file
some_file\nwith\nnewlines
'some_file with spaces'

$ for f in $(ls); do
  if [ -d "$f" ]; then
    echo "Directory: $f"
  elif [ -f "$f" ]; then
    echo "Regular file: $f"
  else
    echo "Not recognized: $f"
  fi
done
Directory: some_dir
Regular file: some_file
Not recognized: some_file\nwith\nnewlines
Not recognized: 'some_file
Not recognized: with
Not recognized: spaces'
```

  Oh, some deranged lunatic has put spaces and newlines in the filenames. Since `IFS` is how POSIX says we should separate fields, spaces will always mangle command output on first contact with the shell's parser. Traditionally, the approach has been to simply change `IFS` to fit the shape of the data. Common solutions in `bash` look something like:
```bash
while IFS= read -r f; do
  echo "got: $f"
done < <(ls)
```
  But even this falls apart, since newlines are `read`'s default delimiter, and there's a file with newlines in the name. The output produced looks like this:

```bash
got: some_dir
got: some_file
got: some_file
got: with
got: newlines
got: some_file with spaces
```

  `ls` in this case puts each filename on its own line, which makes this problem quite complicated. It's a classic problem, the data contains the field separator. Of course, `ls` does have the `-Q` flag with `--quoting-style=shell-escape` to make it render as `'some_file'$'\n''with'$'\n''newlines'`, but more on that later. The problem isn't `ls` specifically, it's the fact that this pattern of output is very common among unix tools, and the programs that produce it often don't have a `-Q` escape hatch like `ls` does.

  This is a very common problem in shell scripting. It's the entire reason why you have to put quotes around your variables and command substitutions *nearly every time*. Once text is emitted to stdout, the intended structure of that data will not survive first contact with the shell's expansion and word splitting logic. It either gets turned into one big glob of text by getting stuffed into quotes, or gets sliced to pieces by the shell's word splitting. There's no inbetween.

  There can be no doubt that this is a genuine problem with shell scripting.

  `nushell` is an interesting attempt to provide a solution to this problem by making a radical architectural decision: it gets rid of the bytes-between-processes concept entirely. Where `bash`'s pipe is two OS processes talking through `pipe(2)`, `nushell`'s pipe is internal function composition between Rust functions running in the same process. Each stage produces and consumes typed, owned Rust values.

  The idea is executed brilliantly, and it actually works very well. There's no field-boundary problem when fields are an intrinsic property of the data structure. `ls | where name =~ "foo"` does an actual field comparison on the `name` field of each `Record`. The whole `IFS` class of bugs simply cannot exist here. The trade-off shows up at the boundary. The moment you pipe `nushell`'s structured output to an external tool, the type information collapses back into a byte stream. Round-trips through any non-`nushell` program require manually re-parsing the structure on the other side.

  So anyway what does all of this have to do with "quoted data streams"?

  Recently I added two new builtins to `shed`, in `v0.29.0`. These builtins were `quote` and `unquote`. At the time I wasn't really thinking much of these additions, I just thought that providing API access to `shed`'s `expand::shell_quote()` function and quote removal would be somewhat useful for round-tripping data without having to worry about field splitting.

  I was messing around with these builtins and found that the pattern these commands provide is genuinely very useful for maintaining the structure of data. I wrote this function called `split`:
```bash
split() {
  [ "$#" -eq 2 ] || raise "Usage: split <%1> <%2> " 'pattern' 'string'
  [ -z "$2" ] && return
  local pat="$1"
  local parts="$2"
  local part

  # in shed, param expansions return 1 if they dont change the var
  # so we can use them in while loop conditions

  # strip all trailing delimiters
  while parts="${parts%${pat}}"; do :; done

  # attach a delimiter to the end
  parts="${parts}${pat}"

  # run until either of these expansions doesnt strip anything
  while part="${parts%%${pat}*}" && parts="${parts#*${pat}}"; do
    quote "$part";
  done
}
```

  Splitting strings into structured output is a genuinely awkward thing to abstract over in shell scripts. You rarely see clean helper functions for this, because there's no decent way to return multiple values from a shell function. The options are word-split output that breaks on spaces, joined-with-quotes output that has to be re-parsed by the caller, or unsafe `eval`-based name-passing. The result is that splitting logic usually just gets inlined.

  However, the function above emits data using `quote`, inside the while loop at the end there. The result is that individual emissions that would be affected by shell word splitting are *automatically wrapped in single quotes*. And `quote` is also aware of control characters like `\n`, if those appear, it uses ANSI-C quotes (`$'...'`) instead.

  The output looks like this:
```bash
$ var="foo:bar buzz:biz:foo bar biz:bam"

$ split : "$var"
foo
'bar buzz'
biz
'foo bar biz'
bam
```

  If you've spent a lot of time around POSIX shell scripts, I imagine this raised your eyebrows. It certainly raised mine. The problem with this though, is that there's not a clean way to *deserialize* the quoted fields. Which is where the `unquote` builtin comes in. `unquote` is used to cleanly unwrap shell-quoted fields. Using the `-a` flag, we can pack the fields into an array:
```bash
$ split : "$var" | wc -l # should be 5
5

$ split : "$var" | unquote -a fields # packs data into '$fields'

$ echo ${#fields} # should also be 5
5

$ echo ${fields[1]} # this one had spaces
bar buzz
```

  The data's shape is completely preserved, untouched by shell word splitting. Traditionally in POSIX scripting, word splitting is all or nothing. The output of `split` in other shells would have to be handled as one giant blob of text by wrapping the entire output in quotes, or many fragments of split up text. In the function though, we clearly know what the shape of the data is. Every single instance `"$part"` is its own field. With `quote` we can actually tell downstream consumers how the data should be interpreted.

  While I was implementing `quote` and `unquote`, I realized there could be potential for allowing the other builtins to get in on this. I added a `-q` flag to the `read` builtin. Adding a new flag to a command as old as time itself feels a little strange, but it was definitely worth it. Remember the `-Q` thing for `ls` that I mentioned? Here's what that actually does to the `ls` output in that directory from earlier:
```bash
$ ls -Q --quoting-style=shell-escape
some_dir   some_file  'some_file'$'\n''with'$'\n''newlines'  'some_file with spaces'
```

  Nice and quoted. Anyway, that `-q` flag I added to `read` makes the command use shell quoting to delimit fields instead of a delimiter byte. Check this out:
```bash
$ while read -q field; do
      echo "got: $field"
  done < <(command ls -Q --quoting-style=shell-escape)
got: some_dir
got: some_file
got: some_file
with
newlines
got: some_file with spaces
```

  The file with newlines and the file with spaces were both properly interpreted as a single field each. In order to get similar quote handling in bash, you would need something like:
```bash
$ while IFS= read -r line; do
      eval "field=$line"
      echo "got: $field"
  done < <(command ls -Q --quoting-style=shell-escape)
```

  Which "works", but using `eval` for stuff like this is ugly at best and dangerous at worst. It's not a good habit. The alternative to potentially allowing arbitrary code execution on your machine is writing a shell quote parser in bash, and like, no.

  It's interesting that this shell-quoting pattern actually appears in many places in the Unix ecosystem. `printf %q`, `set -x` output, `declare -p`, `${var@Q}`, they all emit the same shell-escaped quoting. The encoding has been around forever. But there's just no clean way to actually *decode* it in a way that doesn't involve literally executing code with a shell.

  Once we have both producers *and* consumers of this "shell quoting protocol", the next question is whether or not we can actually use this pattern to effectively communicate with outside programs. As it turns out, adding a `-0` flag to `unquote` makes this not only possible but *ergonomic*. Take a look at this:
```bash
$ var=$'foo:bar\n buz\nz:biz:foo b\nar biz:b\nam' # ick

$ split : "$var" | unquote -0 | xargs -0 -I {} echo $'----\nfield:' {}
----
field: foo
----
field: bar
 buz
z
----
field: biz
----
field: foo b
ar biz
----
field: b
am
```
  The field structure survives the `split` call, is unwrapped and formatted by `unquote`, delimiting by `\0`, then passed to `xargs -0` in a way it knows how to read. The `quote` builtin effectively gives us a way to actually *return structured data* from a shell function, and `unquote` gives us options for formatting that data. For instance, the `-s` flag lets you give an arbitrary field delimiter, instead of `-0` for the null byte. Here's `unquote` generating CSV with it:
```bash
$ split : "$var" | unquote -s ','
foo,bar buzz,biz,foo bar biz,bam
```

  The format here isn't necessarily breaking new ground, this shell-quoting-input pattern has existed for a while, but it doesn't feel like it has really been formalized. All of the code I've seen that makes use of shell-quoted output from stuff like `printf %q` uses bespoke handling methods or relies on `eval` to access the quote removal that the shell never bothered to provide first-class access to.

  It's my opinion that this "serialize fields with `quote`, deserialize with `unquote`" pattern could become a central primitive for `shed`'s scripting model. Implementing more first-class support for shell-quoted input/output like `read` got with the `-q` flag seems like a natural next step for this, and I imagine many of the library functions like `split` that I end up writing for `shed` will most likely end up using `quote` to emit data.
