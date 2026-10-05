---
layout: post
title: Shell Variables Are Files Now
date: 2026-10-05
---

Sort of.

A few posts ago I wrote about `shed`'s [*Virtual File Descriptor Table*](https://km-clay.github.io/2026/09/11/fd-table.html). When describing the idea, one of the things I said about its capabilities was that "we can now create entirely new types of files". Most of the leverage I have gotten from that fact has been internal - for stuff like threaded pipelines and whatnot. I recently added a user-facing application of this, however.

In `shed`, shell variables can now be used in I/O redirections as both targets *and* sources, using this `@var` syntax:

```sh
$ echo foo >@var

$ echo $var
foo

$ cat <@var | { echo "var contains: $(thru)"; }
var contains: foo

$ { echo "error!" >&2; echo "output!"; } >@out 2>@err

$ echo $err
error!

$ echo $out
output!
```

Magic!

This isn't super crazy on its own, though. Skeptical readers may have already realized that the first example is just trivial command substitution:

```sh
$ echo foo >@var

$ echo $var
foo

$ unset var

$ var=$(echo foo)

$ echo $var
foo
```

There is one big difference though.

```sh
$ var=$(echo foo)

$ len -b "$var"
3

$ unset var

$ echo foo >@var

$ len -b "$var"
4
```

Different byte lengths! What happened?

Well, POSIX command substitution is actually a *lossy capture*. And it's not even a bug or an oversight, it's *mandated by the specification*:

> [POSIX Shell Command Language § 2.6.3](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_06_03)
>
> The shell shall expand the command substitution by executing command in a subshell environment (see [Shell Execution Environment](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_12)) and replacing the command substitution (the text of command plus the enclosing "$()" or backquotes) with the standard output of the command, ***removing sequences of one or more &lt;newline&gt; characters at the end of the substitution.***

Newline characters (`\n`) are stripped from the output. This is so that something like `"The $(echo 'quick') brown fox"` resolves to

```
"The quick brown fox"
```

and not something like

```
"The quick
brown fox"
```

This is very reasonable when your shell is only capable of working with text, but *unacceptable* when your shell can operate on raw binary and needs some kind of lossless command output capture.

The POSIX specification notably provides no options for this really, apart from pipelines themselves, which is why `thru` gained the `-v var` flag. Then `str trim`, `str clip`, `pop`, and `fpop` also all gained this same exact flag. Five builtins suddenly acquiring the same behavior is a strong signal that the behavior should belong to *the shell itself*. Once I had implemented `>@var`, all five of them dropped the flag, losing zero utility in the process.

So the obvious next question is "what does `>@var` buy us that `-v var` doesn't"? Well, the biggest thing is that the capability actually extends to *all commands*, not just builtins. *Forked external commands can set shell variables*.

```sh
$ date "+%x" >@date

$ echo $date
10/05/2026
```

Magic!!

And of course, the bytes read into the variable are untouched.

### Null Bytes

Informed readers may be skeptical about one thing though: how does this handle the NUL (`\0`) byte? In general (apart from `zsh`), POSIX shell variables *cannot hold an interior NUL byte*. `shed`'s string type, however, is literally just a fancy byte array. It has a known length attached to it, so it doesn't use NUL bytes as a terminator, it treats them like any other byte. This fact basically closes the loop on byte operations; producers can losslessly write directly into shell variables with `>@var` or `>>@var`, and a consumer can read those bytes directly with `<@var`.

Here's what happens when you try to store a string with a null byte in a variable in `bash`:

```sh
[pagedmov@tourian:~]$ q=$(printf 'a\0b');
bash: warning: command substitution: ignored null byte in input

[pagedmov@tourian:~]$ echo ${#q}
2

[pagedmov@tourian:~]$ echo "$q"
ab

[pagedmov@tourian:~]$ v=$'a\0b' # this one doesn't even warn, it just truncates silently

[pagedmov@tourian:~]$ echo ${#v}
1

[pagedmov@tourian:~]$ echo $v
a
```

and the same thing in `shed`:

```sh
┏━ pagedmov@tourian
┣━━ ~/
┗━ $ q=$(printf 'a\0b')

┏━ pagedmov@tourian
┣━━ ~/
┗━ $ echo ${#q}
3

┏━ pagedmov@tourian
┣━━ ~/
┗━ $ printf '%s' "$q" | xxd
00000000: 6100 62                                  a.b

┏━ pagedmov@tourian
┣━━ ~/
┗━ $ v=$'a\0b'

┏━ pagedmov@tourian
┣━━ ~/
┗━ $ echo ${#v}
3

┏━ pagedmov@tourian
┣━━ ~/
┗━ $ printf '%s' "$v" | xxd
00000000: 6100 62                                  a.b
```

`xxd` reads `6100 62` for both cases. The null byte survived both storage and retrieval. Important to note that this isn't necessarily breaking new ground; there *are* other shells that also see the value in being byte-exact when it comes to variable storage. For instance, `zsh`:

```sh
pagedmov@tourian:~/projects/fern/ > v=$'a\0b'
pagedmov@tourian:~/projects/fern/ > echo ${#v}
3
pagedmov@tourian:~/projects/fern/ > printf '%s' "$v" | xxd
00000000: 6100 62                                  a.b
```

### Channel Splitting

The opening example demonstrates a killer use-case for this feature:

```sh
$ { echo "error!" >&2; echo "output!"; } >@out 2>@err

$ echo $err
error!

$ echo $out
output!
```

As far as I know, multi-channel output capture like this isn't representable in the POSIX shell language. Command substitution only reads from one stream. The only way to keep err+out capture through a command substitution is by duping stderr onto stdout and contaminating both streams. Thus, the only way to keep them separate *and* capture both is by using tempfiles, which is never a clean solution and forces assumptions about the script's environment (able to write files, `mktemp` in PATH, etc).

### Persistent Variable Redirection

Variables can also now be *opened on file descriptors* using `exec`:

```sh
$ echo $'foo\nbar\nbiz' >@out

$ echo "$out"
foo
bar
biz


$ exec 5<@out

  $  while thru --until $'\n' >@line <&5; do
2 |      echo "got line: $line"
3 |  done
got line: foo
got line: bar
got line: biz
```

Though, this is basically where the "sort of" statement from the start of the post comes in. Under the hood, variable redirections use this system to actually work:

1. on creation, feed variable content into a buffer
2. builtins read from that buffer, but if the redirection is on an external command, it gets promoted to an actual os-level file descriptor:
    * vars opened for *reading* become a `memfd` on Linux and `tempfile` on BSD/macOS. (same system as heredocs actually)
    * vars opened for *writing* become a write-only pipe fd that drains into an internal buffer. A pipe rather than a file, so that writers get backpressure - a runaway producer blocks and then dies on `EPIPE` instead of materializing gigabytes first.
3. of course, once the content is fed into this buffer, the actual variable itself held by the shell state no longer has any bearing on the value held in the buffer/file descriptor.
4. on destruction of a variable fd *opened for writing*, the variable it targets is clobbered by the content of the buffer/fd

This has some implications for `exec` called on variable redirections.

Variables that are opened for writing do not have their contents changed *until the file descriptor is closed*.

```sh
$ echo 'foo bar biz' >@out

$ exec 5>>@out      # open for appending

$ echo " fizz " >&5 # append some text

$ echo "$out"       # not committed yet!
foo bar biz


$ exec 5>&-         # commits changes

$ echo "$out"       # committed now!
foo bar biz
 fizz
```

The content of the actual `$out` variable itself remains the same until the close. This is basically the biggest 'gotcha' for working with long-lived var redirects. A simple way to manage this behavior is to use `defer`:

```sh
   $ echo -n 'foo bar biz' >@out
 2 | {
 3 |     exec 5>>@out    # open for appending
 4 |     defer exec 5>&- # guarantees the close
 5 |
 6 |     echo -n " fizz" >&5
 7 |     echo -n " boz" >&5
 8 |     echo -n " fazz" >&5
 9 | }
10 |
11 | echo "$out"
foo bar biz fizz boz fazz
```

In order to prevent blowing up your ram with insanely large command output captures, the output capture respects the value of `shopt core.max_read_limit`. A single variable redirection (one shot with `>@var` or when `exec 5>&-` is called) will always truncate at the size held in that shopt.

---

There's also some things to keep in mind with the specific case of *compound commands in pipelines*. Take this command for example:

```sh
printf x | { cat >@v; } | cat
```

Based on observations from the other uses of variable redirection in this post, one would assume that `$v` will hold the value `x` when this pipeline returns, but that is not the case. This is an issue with the time-of-construction for the variable redirection; by the time it has been applied, the brace group it is inside of *has already forked*, meaning only the child process will see the new variable. Contrast this with a very-similar-looking command:

```sh
printf x | { cat; } >@v | cat
```

In *this* case, `$v` actually is correctly set to `x` in the parent, and the change persists once the pipeline ends. This is because the redirection is constructed and applied *before* the brace group forks. This weirdness basically only happens under the following specific conditions:

1. The redirection is inside of a compound command like a brace group or if statement
2. The compound command is a stage in a pipeline

Note that where the compound sits in the pipeline makes no difference; it fails at the end just the same as in the middle:

```sh
$ printf x | { cat >@v; }

$ echo "[$v]"
[]
```

In practice, I can't personally see this ever causing issues. The main thing to remember is just to keep variable redirections outside of compound commands when the compound is a pipeline stage. A correct pattern would be something like this:

```sh
printf x | { echo "reading!" >&2; cat; } 2>@v | cat
```

as opposed to this:

```sh
printf x | { echo "reading!" >@v; cat; } | cat
```

Essentially pointing the inner commands at the channels that the compound command's redirection will receive.

Also, the variable name used in the redirections can *index arrays*

```sh
$ foo=()

$ for i in $(seq 0 5); do
2 |      echo "$i" >@foo[i]
3 |  done

$ echo "${foo[@]}"
0
 1
 2
 3
 4
 5

$ declare -A foo

$ echo "bar" >@foo[one]

$ echo "biz" >@foo[two]

$ echo "baz" >@foo[three]

$ echo $foo
one=$'bar\n' two=$'biz\n' three=$'baz\n'
```

Note that the byte structure of the input is maintained; the captured outputs retain their newline.

### Example Usage

Here's a sample function that makes use of the variable-as-fd feature; a file type detector. With `thru >@_buf` and `exec {in}<@_buf`, the input to the function becomes arbitrarily seekable, enabling us to parse these binary headers pretty effortlessly:

```sh
# filetype detector
# reads raw binary headers from the file and deduces filetype
filetype() {
    thru >@_buf        # buffer input
    exec {in}<@_buf    # open buffered input on fd $in
    defer exec {in}>&- # defer closure of fd $in

    # read 16 bytes from input into the $header variable
    thru -T 16 <&{in} >@header
    case "$header" in
        $'\x89PNG\r\n\x1a\n'*)      echo png ;;
        $'\xff\xd8\xff'*)           echo jpeg ;;
        'GIF87a'*|'GIF89a'*)        echo gif ;;
        '%PDF-'*)                   echo pdf ;;
        'PK'$'\x03\x04'*)           echo zip ;;
        $'\x1f\x8b'*)               echo gzip ;;
        'BM'*)                      echo bmp ;;
        $'\x00\x00\x01\x00'*)       echo ico ;;
        'SQLite format 3'$'\x00'*)  echo sqlite ;;
        '<svg'*|'<?xml'*)           echo svg-or-xml ;;
        $'\x7fELF'*)
            # its an elf executable
            # seek back to byte 4, read a byte
            seek $in 4
            thru -T 1 <&{in} >@byte
            [ "$byte" = $'\x02' ] && echo elf64 || echo elf32
        ;;
        'RIFF'*)
            # a RIFF file format
            # seek back to byte 8 and read 4 bytes
            seek $in 8
            thru -T 4 <&{in} >@format
            printf 'riff/%s\n' "$format"
        ;;
        *)
            # tar or something?
            # check for the pattern 'ustar' at byte offset 257
            seek $in 257
            thru -T 5 <&{in} >@ustar
            if [ "$ustar" = ustar ]; then echo tar; else echo unknown; fi
        ;;
    esac
}
```

```sh
   $ filetype < ~/shed_preview.png
 2 | filetype < ~/pictures/HTDN0vZWwAASrYr.jpg
 3 | filetype < ~/Downloads/how-you-felt-hero.gif
 4 | filetype < ~/repos/readline/doc/readline_3.pdf
 5 | filetype < ~/tetrio_touhou.zip
 6 | filetype < /nix/store/bpkrhiryx7dkahy8qbwk9qnhkk4bcjcy-crate-futures-core-0.3.34.tar.gz
 7 | filetype < ~/repos/oils/Python-2.7.13/Lib/test/testtar.tar
 8 | filetype < ~/misc/eb_sfx/'042 Hypnosis.wav'
 9 | filetype < ~/.sysflake/assets/wallpapers/selective_color/osaka.webp
10 | filetype < /nix/store/mhk80g6y2bi8h56ljgrqw1r1amj6qvd7-obsidian-logo-gradient.svg
11 | filetype < /nix/store/s5q6s4vj4cr9b0k0id189yay0ajhd4ia-vlc-3.0.24/share/vlc/vlc.ico
12 | filetype < ~/media/games/fourtris/buttons.bmp
13 | filetype < ~/.librewolf/x3bu4pxn.default/storage.sqlite
14 | filetype < ~/projects/fern/target/debug/shed
png
jpeg
gif
pdf
zip
gzip
tar
riff/WAVE
riff/WEBP
svg-or-xml
ico
bmp
sqlite
elf64
```

Magic!!!

This script *can* be written using standard POSIX shell scripting, if you're OK with making several concessions:

* need to do all of the seeking using `dd`, which assumes that `dd` is available in the environment
* stdin must be dumped to a tempfile so that `dd` has something to seek.
* each header must be piped through `od` and `tr` to hex-encode it (ugly and also two more environment assumptions)
* you write your magic numbers like `53514c69746520666f726d6174203300`. (ew)

It works, but it also costs four forked processes per read and the patterns are illegible. And the only reason why you're encoding to hex is because the shell can't hold the bytes itself in the first place. `shed` is able to skip all of this for free.

### Conclusion

One thing that has stood out to me while working on this and the other byte-operation features has really been how much the POSIX specification *assumes* that shells *only work on text*. `${#var}` prints character counts and not byte counts, command substitution strips newlines, and in some implementations even strips interior null bytes (e.g. `bash`), the list goes on.

When I first started this foray it kind of felt like the structure of shell scripting itself was fighting it, but as I implemented the lower-order byte primitives like `thru` and `readint`/`writeint` (and now `>@var` redirection), it's really shown that shells were always capable of operating in this domain. Not only are they capable of it, there's basically no reason not to capitalize on that capability. Operation on raw bytes is strictly a superset of operation on text - text is literally just bytes after all - so any shell that is capable of operating on bytes is capable of operating on text using the same machinery.
