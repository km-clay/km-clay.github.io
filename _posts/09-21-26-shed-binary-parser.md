---
layout: post
title: Writing a Binary Parser in shed
date: 2026-09-21
---
Recently I started playing the guitar. I've been looking for stuff to try playing, and thought it would be cool to try some of the songs from the classic windows-era Touhou games. The albums for these games' soundtracks *are* available on spotify, but I wanted to have the actual files handy so I could do stuff like muting specific instruments with Demucs to turn them into backing tracks. I decided to try my hand at extracting the music from Touhou 7: Perfect Cherry Blossom to nail down a method before doing the other games too.

Luckily I happen to have the files for these games downloaded locally, the only issue is that the soundtracks are baked into the game's `.dat` files, which are basically just binary blobs. No problem, though; the Touhou community maintains a program called the Touhou Toolkit that can be used to decompile the game data!

I cloned the repository for this program and built it, only to find out that `thtk` only operates on the main `th07.dat` file, and not the music-holding `thbgm.dat` file (could be wrong about that but idk). I could use `thtk` to extract a file called `thbgm.fmt` from the main `.dat` blob, but that's as far as `thtk` would be able to take me on this. `thbgm.fmt` is *another* binary blob that essentially contains the slicing instructions that allow the game to cut the `thbgm.dat` blob file into several `.wav` files. Encoded in binary, the `thbgm.fmt` file contains records that look like this:


| offset | size | field |
|--------|------|-------|
| 0x00 | 16 | song name (ASCII) |
| 0x10 | 4 | byte offset into `thbgm.dat` (uint32) |
| 0x14 | 4 | ??? (uint32) |
| 0x18 | 4 | ??? (uint32) |
| 0x1C   | 4 | length in bytes (uint32) |
| 0x20 | 18 | WAVEFORMATEX (.wav headers) |
| 0x32 | 2 | padding |


These records each take up 52 bytes each, and therefore we can just slice the file 52 bytes at a time and parse the chunks according to the table above. Once we've decoded all of the fields, we'll have all of the information necessary to start slicing the `thbgm.dat` file and getting the songs out of it. Cool!

While I was weighing my options for how to approach actually stepping through this file, I had an interesting idea: could I use `shed`'s scripting language to write this binary parser?

now, most if not all existing shells would really choke on a script of this nature, for a few reasons; some obvious, and some obscure:
1. The NUL(`\0`) byte: shell variables *cannot* hold a nul byte in traditional shells. This is because their backing type is C strings, so the NUL byte signals the end of the string.
2. Command substitution: it strips trailing newline bytes (`\n`). This is byte 10, or `0x0a`. If the bytes you capture happen to end in `0x0a`, that byte will mysteriously vanish.
3. Random access into a file is not easy. The best most shells can do to move a file descriptor's cursor manually is using short-reads with `dd`, which is about as elegant as slicing bread with a chainsaw, and requires an entire fork per read.

And probably more that I *don't* know of. Operation on raw binary is actually unspecified by POSIX, so most shells have reasonably just decided to discard nulls and treat everything as text.

All of this makes working with raw binary in a shell a real struggle. But it occurred to me that `shed` is actually faced with exactly *zero* of these problems thanks to a refactor a while ago that migrated `shed`'s I/O handling to use its own string type - one that holds the data as raw bytes rather than as strings.

This design actually does allow shed to operate on and store raw byte streams that *can* hold interior null bytes. On top of this, shed has the `seek` builtin which allows for arbitrarily moving a file descriptor's cursor using the `lseek()` system call. And on top of even *that*, the `thru` builtin allows for limiting reads to exact byte counts. `thru -L 52` in a loop slices the input stream into 52 byte chunks for free.

When you put all of this together, parsing a raw binary file format manually actually becomes quite simple. Though, there were a few missing components that I needed in order to fully realize this project; the first of which was finding a way to signal when we are out of data when using `thru` to step through the input.

At the time, `thru` always returned 0 upon completing a read. This is technically correct behavior, so instead of changing it, I added a new flag: `--report-eof`/`-E`. This flag makes `thru` return 1 if its internal read loop ever hits EOF. This allows us to easily write a while loop that uses `thru` to step through the input one chunk at a time: `while thru -E -L 52 <&6 > chunk.bin; do ...`.

There was also a POSIX-specified `printf` behavior that `shed` had not implemented yet; when the leading character of an argument is a double or single quote, the next char is converted to its raw byte value. This is very useful when extracting null-terminated strings from binary data - if the resulting byte value is 0, we know to stop.

And one last necessary addition: `read -N`. `shed` already implements the `-n` flag which lets you read a specific number of bytes, but it still performs word splitting and respects delimiters. `-N` is an even more forceful version of this that lets you specify an exact number of bytes to read, while also ignoring delimiters and not performing `$IFS` word splitting of any kind. The bytes read are processed and passed through as purely opaque data.

With all of this in place, we can start writing our binary parser. For this case, we have redirected `thbgm.fmt` to fd 6 and are writing the chunks directly to a binary file called `chunk.bin`. Shed's `defer` keyword also makes management of these resources simple:

```sh
{ # brace groups can be used to scope 'defer' calls
    defer exec 6>&-
    defer rm chunk.bin

    exec 6< thbgm.fmt
    # thru -E lets us use it as a while loop condition, sorta like the `read -r line` idiom
    while thru -E -L 52 <&6 > chunk.bin; do
        # ... use chunk.bin ...
    done
}
```

Perfect, we now have our chunk slicing logic in order. Now we just need something that slices the data in `chunk.bin` into something usable. At this point, the contents of `chunk.bin` reflect the table from earlier. All we need to do now is slice the data according to that table and we will have our song extraction instructions for `thbgm.dat`.

But how can we do that exactly? This is the part where the pipeline logic starts to get really involved, and `thru`'s `-L` flag really starts to pay off in spades here.

With these helpers:
```sh
# reads EXACTLY 1 byte
read_byte() { IFS= read -N 1 "$1"; }

read_n() {
    # build a number, little endian
    local -i num=0;
    local -i bit=0;
    local -i n

    while read_byte b; do
        n=$(printf '%d' "'$b") # n is now a uint8
        num=$(( num | (n << bit) )) # OR
        bit+=8 # we filled 8 bits, shift by 8
    done
    echo "$num"
}
```

The final parser loop looks like this. Notice the strangely tree-like structure of the `thru` calls:
```sh

data=()

while thru -E -L 52 <&6 > chunk.bin; do
    # make sure the chunk is the right size
    [ "$(stat -c '%s' chunk.bin)" -eq 52 ] || break

    # write the chunk into this brace group
    thru chunk.bin | {
        # each thru call now consumes the input incrementally
        # each one taking an exact number of bytes
        thru -L 16 | {
            # name field
            name=""
            while read_byte char; do
                byte=$(printf '%d' "'$char")
                [ "$byte" -eq 0 ] && break
                name+="$char"
            done
            push data "$name"
        }
        thru -L 4 | {
            # offset field
            push data "$(read_n)"
        }
        thru -L 8 > /dev/null # unidentified garbage
        thru -L 4 | {
            # length field
            push data "$(read_n)"
        }

        # .wav header junk
        for chunk in 2 2 4 4 2 2 2; do
            thru -L $chunk | {
                push data "$(read_n)"
            }
        done
    }

    # emit the structured data
    quote "${data[@]}"
    data=()
done
```

With this, we now have a function that correctly extracts all of the data from `thbgm.fmt`. Here's what that data looks like when it's structured:


| name | offset | length | format | channels | sample rate | byte rate | align | samplebits | cbsize |
| ---- | ------ | -------- | ------ | -------- | ---------- | -------- | ----- | ---------- | ------ |
| th07_01.wav | 16 | 16545568 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_02.wav | 16545584 | 15499264 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_03.wav | 32044848 | 12287996 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_04.wav | 44332844 | 24640140 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_05.wav | 68972984 | 9926136 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_06.wav | 78899120 | 22018560 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_07.wav | 100917680 | 31686656 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_08.wav | 132604336 | 43193344 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_09.wav | 175797680 | 24899328 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_10.wav | 200697008 | 26254336 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_11.wav | 226951344 | 19758080 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_12.wav | 246709424 | 10138624 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_13.wav | 256848048 | 25412608 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_13b.wav | 282260656 | 16108800 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_14.wav | 298369456 | 11411712 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_15.wav | 309781168 | 17170432 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_16.wav | 326951600 | 25173760 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_17.wav | 352125360 | 37693568 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_18.wav | 389818928 | 25173760 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |
| th07_19.wav | 414992688 | 29523968 | 1 | 2 | 44100 | 176400 | 4 | 16 | 0 |


Very nice. Now we have all of the info we need to start extracting songs from `thbgm.dat`. We are now going to be using this information to slice raw bytes out of `thbgm.dat` and use those bytes to hand-write some `.wav` files. This is the part where the `seek` builtin shows up. For the sake of brevity, I packed the code block above into this `parse_fmt` shell function. This is what we're starting with:

```sh
parse_fmt thbgm.fmt | {
    # helpers for writing ints
	put_byte() { printf '%b' "\\0$(printf '%03o' $(( $1 & 0xff )))";   }
	put_le16() { for i in 0 8;       do put_byte $(( $1 >> i )); done; }
	put_le32() { for i in 0 8 16 24; do put_byte $(( $1 >> i )); done; }

    exec 5< thbgm.dat # open thbgm.dat on fd 5
    defer exec 5>&-
    while read -r name off size tag ch rate avg align bits cbsize; do
        # ...
    done
}
```

The `while` loop will decode the data into the field variables, now we just need to use it to manually write the `.wav` file. `.wav` files are a specific form of RIFF (Resource Interchange File Format), which is a generic little-endian container that Microsoft and IBM defined around 1991. RIFF's entire model for structuring data is chunks, and without getting too involved, the general format for `.wav` files specifically is this:


| data | type |
| ---- | ---- |
| "RIFF" | RIFF chunk id |
| <36 + data size> | size of entire chunk from this point |
| "WAVE" | form type |
| "fmt " | subchunk id (note the trailing space) |
| <16> | fmt body size (16 in this case) |
| audio format | uint16 LE |
| channels | uint16 LE |
| sample rate | uint32 LE |
| byte rate | uint32 LE |
| block align | uint16 LE |
| bits/sample | uint16 LE |
| "data" | subchunk id |
| &lt;data size&gt; | size of data chunk |
| &lt;data&gt; | raw chunk data (song bytes) |


We have to synthesize one `.wav` file per record, using this binary format. With those helpers from the last code block, this becomes easy:
```sh
parse_fmt thbgm.fmt | {
    # helpers for writing ints
    put_byte() { printf '%b' "\\0$(printf '%03o' $(( $1 & 0xff )))";   }
    put_le16() { for i in 0 8;       do put_byte $(( $1 >> i )); done; }
    put_le32() { for i in 0 8 16 24; do put_byte $(( $1 >> i )); done; }

    exec 5< thbgm.dat # open thbgm.dat on fd 5
    defer exec 5>&-
    while read -r name off size tag ch rate avg align bits cbsize; do
        {
            # emit the file header first
            printf "RIFF"
            put_le32 "$(( 36 + size ))"
            printf "WAVE"
            printf "fmt "
            put_le32 "16"
            put_le16 "$tag"
            put_le16 "$ch"
            put_le32 "$rate"
            put_le32 "$avg"
            put_le16 "$align"
            put_le16 "$bits"
            printf 'data'
            put_le32 "$size"

            # now slice the data out of thbgm.dat
            # seek to exactly '$off' bytes
            seek 5 $off > /dev/null
            # read exactly '$size' bytes
            thru -L $size <&5
        } > "out/$name" # write the file
    done
}
```

And that's literally it. We have our `.wav` files now:
```sh
ls -la out

total 434152
drwxr-xr-x 2 pagedmov users     4096 Sep 21 13:57 .
drwxr-xr-x 3 pagedmov users     4096 Sep 21 20:32 ..
-rw-r--r-- 1 pagedmov users 16545612 Sep 21 19:10 th07_01.wav
-rw-r--r-- 1 pagedmov users 15499308 Sep 21 19:10 th07_02.wav
-rw-r--r-- 1 pagedmov users 12288040 Sep 21 19:10 th07_03.wav
-rw-r--r-- 1 pagedmov users 24640184 Sep 21 19:10 th07_04.wav
-rw-r--r-- 1 pagedmov users  9926180 Sep 21 19:10 th07_05.wav
-rw-r--r-- 1 pagedmov users 22018604 Sep 21 19:10 th07_06.wav
-rw-r--r-- 1 pagedmov users 31686700 Sep 21 19:10 th07_07.wav
-rw-r--r-- 1 pagedmov users 43193388 Sep 21 19:10 th07_08.wav
-rw-r--r-- 1 pagedmov users 24899372 Sep 21 19:10 th07_09.wav
-rw-r--r-- 1 pagedmov users 26254380 Sep 21 19:10 th07_10.wav
-rw-r--r-- 1 pagedmov users 19758124 Sep 21 19:10 th07_11.wav
-rw-r--r-- 1 pagedmov users 10138668 Sep 21 19:10 th07_12.wav
-rw-r--r-- 1 pagedmov users 16108844 Sep 21 19:10 th07_13b.wav
-rw-r--r-- 1 pagedmov users 25412652 Sep 21 19:10 th07_13.wav
-rw-r--r-- 1 pagedmov users 11411756 Sep 21 19:10 th07_14.wav
-rw-r--r-- 1 pagedmov users 17170476 Sep 21 19:10 th07_15.wav
-rw-r--r-- 1 pagedmov users 25173804 Sep 21 19:10 th07_16.wav
-rw-r--r-- 1 pagedmov users 37693612 Sep 21 19:10 th07_17.wav
-rw-r--r-- 1 pagedmov users 25173804 Sep 21 19:10 th07_18.wav
-rw-r--r-- 1 pagedmov users 29524012 Sep 21 19:10 th07_19.wav
```

The songs end abruptly since the game itself handles looping, but other than that these can be opened by VLC and sound perfect. As a verification step, take a look at the size field for every file. Each one is exactly `44 + length` bytes in size, which is the header we constructed, followed by the slice we took from `thbgm.dat`. The combo of `seek` and `thru -L` allows for very high-precision slicing of file content.

This project ended up exposing a fair amount of implementation blindspots and actual bugs. The `-E` flag for `thru` was only conceived because I needed a stop signal for looped, fixed-size reads. For constructing numbers, I needed to add the posix `printf` leading quote thing that converts chars into raw bytes. `read` was also given the `-N` flag for exact raw byte counts (used in the `read_byte` helper). Creating a bunch of vars with interior nulls also exposed a lot of C-FFI handling errors, since the code was assuming no interior nulls even though the architecture supported it.

On top of this, writing the parser also inspired a new script diagnostic feature: fork tracing. Every command in this parser is actually a `shed` builtin - zero external commands were used to perform this operation. `shed`'s execution logic has heavily optimized builtin-only pathways, with the main goal being reducing fork overhead by executing builtins in-process, even in pipelines. Using `shopt core.fork_trace=true`, `shed` now prints diagnostics for scripts and commands that shows where and how many forks occurred in a given execution:

```sh
profile: 4 fork points
    ╭─[ repl_entry #3:3:8 ]
    │
  2 │ ╭───▶ {
  3 │ │         defer exec 6>&-
    │ │               ────┬────
    │ │                   ╰────── forked here, command
    ┆ ┆
 19 │ │         exec 6< thbgm.fmt
    │ │         ────────┬────────
    │ │                 ╰────────── forked here, command
    ┆ ┆
 54 │ ├───▶ } | {
    │ │         ▲
    │ ╰───────────── compound, 2 fork points
    │           │
    │   ╭───────╯
 55 │   │       defer exec 5>&-
    │   │             ────┬────
    │   │                 ╰────── forked here, command
 56 │   │       exec 5< thbgm.dat
    │   │       ────────┬────────
    │   │               ╰────────── forked here, command
    ┆   ┆
 82 │   ├─▶ }
    │   │
    │   ╰─────── compound, 2 fork points
────╯
```
<sub>note: `exec` is a builtin, but it always forces a fork in pipelines since it alters process state</sub>

Useful for finding spots in your scripts where you could reasonably cut down on system time by using a builtin instead of an external command.

I'm really pleased with how gracefully `shed` is handling intense workloads like this. It's gotten to the point where I'm now actually writing all of my personal shell scripts using shed instead of bash, because the language features and builtins are just that ergonomic to work with. I'm also kind of surprised at how I'm still finding new ways to put shed's builtins together - the combination of `seek` and `thru -L` for slicing byte ranges out of files is something I never would have even considered without this mini-project. Not to mention just `thru -L` itself; I hadn't even used it before today, but it turns out it can actually be used essentially as a "valve" for your pipelines, allowing you to direct I/O with unnaturally fine-grained control.

In general, this opens up a lot of doors for what I consider to be feasible for a shell script to manage. If the shell language can escape from the realm of plaintext, it kind of feels like anything is possible at this point.
