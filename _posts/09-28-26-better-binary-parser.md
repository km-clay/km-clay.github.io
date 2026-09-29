---
layout: post
title: Writing a Better Binary Parser in shed
date: 2026-09-28
---

In my previous post, I wrote about a [binary parser](https://km-clay.github.io/2026/09/21/shed-binary-parser.html) that I wrote using nothing but `shed`'s scripting language and builtin commands. That project actually turned out to be somewhat pivotal for realizing the direction that I would like for `shed` to take. In the conclusion of that post, I mentioned feeling like anything could be possible for a shell that is capable of working with bytes instead of chars. After a while of sitting on that, I decided that `shed` *should* be able to cleanly operate on the byte level. Because why not

A scripting language that can exist on a lower order than text streams, and actually operate on the substrate of the operating system itself (raw bytes), would probably stop being a "shell scripting language" and become something closer to a "Unix scripting language", and that sounds like a worthy goal. Completely plausible as well, given how readily `shed`'s architecture has accepted the primitives other shells seem to avoid (`poll`, `sock`, `listen`, etc).

I took another look at the final script (expanded upon since my last post) that I had used for actually extracting the song files:

```sh
parse_fmt() {
    defer rm chunk.bin
    defer exec 5>&-

    data=()

    read_byte() { IFS= read -N 1 "$1"; }

    read_n() {
        num=0; place=1
        while read_byte b; do
            n=$(printf '%d' "'$b")
            num=$(( num + n * place ))
            place=$(( place * 256 ))
        done
        echo "$num"
    }

    exec 5< "$1"
    while thru -E -L 52 <&5 > chunk.bin; do
        [ "$(stat -c '%s' chunk.bin)" -eq 52 ] || break

        thru chunk.bin | {
            thru -L 16 | {
                name=""
                while read_byte char; do
                    byte=$(printf '%d' "'$char")
                    [ "$byte" -eq 0 ] && break
                    name+="$char"
                done
                push data "$name"
            }
            thru -L 4 | {
                # offset
                push data "$(read_n)"
            }
            thru -L 8 > /dev/null # unidentified garbage
            thru -L 4 | {
                # duration
                push data "$(read_n)"
            }

            for chunk in 2 2 4 4 2 2 2; do
                thru -L $chunk | {
                    push data "$(read_n)"
                }
            done
        }

        echo "${data[@]}"
        data=()
    done
}

write_wav() {
    local dat="$1" dir="$2"
    exec 5< "$dat"
    defer exec 5>&-

    put_byte() { printf '%b' "\\0$(printf '%03o' $(( $1 & 0xff )))"; }
    put_le16() { for i in 0 8;       do put_byte $(( $1 >> i )); done; }
    put_le32() { for i in 0 8 16 24; do put_byte $(( $1 >> i )); done; }

    declare -i total=0
    while read -r name off size tag ch rate avg align bits cbsize; do
        {
            # .wav headers
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

            # seek to exactly '$off' bytes in the file
            seek 5 $off > /dev/null
            # read exactly '$size' bytes from the file
            thru -L $size <&5
        }  > "$dir/$name"
        total+=$size
    done

    printf '%d' "$total"
}

{
    # now we actually do the extractions per game
    [ -n "$THDAT" ] || raise "THDAT is unset"
    defer rm thbgm.fmt
    total=0
    for dir in ~/media/games/touhou/*; do
        declare dat bgm
        for file in "$dir"/*; do
            # get the game and music .dat files
            if [[ "$file" =~ .*/th[0-9]{2}.dat ]]; then
                dat="$file"
                elif [[ "$file" =~ .*/thbgm.dat ]]; then
                bgm="$file"
            fi
        done

        echo "$dat"
        echo "$bgm"

        [ -z "$dat" ] || [ -z "$bgm" ] && continue
        dir="./$(basename "$dir")"
        mkdir -p "$dir"

        # extract
        $THDAT -xd "$dat" thbgm.fmt
        stat thbgm.fmt
        dir_total=$(parse_fmt thbgm.fmt | write_wav "$bgm" "$dir")
        let total=total+dir_total
    done

    echo "done"
    echo "wrote $total bytes"
}
```

This script has three parts:
* `parse_fmt` - parses the `thbgm.fmt` file to get instructions for how to slice the song data out of `thbgm.dat`
* `write_wav` - uses the data provided by `parse_fmt` to write the `.wav` file headers, and slice the bytes out of `thbgm.dat` for writing to the new `.wav` file
* brace group - loops over all of my Touhou game directories, performing this operation on all of them.

In its current state, it accomplishes these goals, but the code is still pretty ugly. It still sort of feels like the shell is contorting itself in some places. I decided to revisit this script to try and infer what `shed` was missing that made it feel that way.

The first thing that stands out:

```sh
read_byte() { IFS= read -N 1 "$1"; }
```

In the previous post, I mentioned adding this `-N` flag for `read`. The idea was sound; I wanted to read a specific number of bytes *into a variable* while not caring about IFS splitting and such. This was necessary because doing something like `var=$(thru)` would clip any bytes that look like newlines at the end of the data. It occurred to me upon further inspection though, that adding a "treat input as raw bytes" mode for `read` was actually an overreach; `read` as a command is *designed* to operate on text streams. Reading raw bytes from stdin was what `thru` was made to do, and `thru` already had a flag specifically for limiting reads.

The real solution was to add a mode for `thru` that reads into a variable, instead of writing to stdout: the `-v` flag. With this, the name field and `read_n` loops could use `thru` instead of `read_byte`:

```sh
read_n() {
    num=0; place=1
    while thru -L 1 -v b; do
        n=$(printf '%d' "'$b")
        num=$(( num + n * place ))
        place=$(( place * 256 ))
    done
    echo "$num"
}

# ...

thru -L 16 | {
    name=""
    while thru -L 1 -v char; do
        byte=$(printf '%d' "'$char")
        [ "$byte" -eq 0 ] && break
        name+="$char"
    done
    push data "$name"
}
```

And the `read_byte` helper could be thrown away entirely. There was still room for improvement though. The name field loop (the bottom one) felt like an easy next target. The entire idea of this is to just stream bytes one at a time until you find a specific byte, then break. For this, I added the `--until` flag for `thru`, which lets you pick a specific byte that makes `thru` stop reading once you encounter it. The name field loop then collapses even further:

```sh
thru -T 16 | {
    # name, null terminated
    thru --until $'\0' -v name
    push data "$name"
}
```

I also had the idea to allow the `push` builtin to receive input from stdin, basically taking the input stream and pushing it to the array. The name field loop finally arrives at this:

```sh
thru -T 16 | {
    # name, null terminated
    thru --until $'\0' | push data
}
```

A single line of code to express "read a null terminated string from raw binary input, and then push it to the 'data' array". `thru` ended up getting a handful more flags besides just this, based on the requirements of this script, but I won't go into too much detail since they aren't used directly in the script. The only relevant thing there is that `-L`/`--limit` was renamed to `-T`/`--take`.

The next thing that kind of smelled was the way I was handling integers:

```sh
# parse_fmt
read_n() {
    num=0; place=1
    while read_byte b; do
        n=$(printf '%d' "'$b")
        num=$(( num + n * place ))
        place=$(( place * 256 ))
    done
    echo "$num"
}
# write_wav
put_byte() { printf '%b' "\\0$(printf '%03o' $(( $1 & 0xff )))"; }
put_le16() { for i in 0 8;       do put_byte $(( $1 >> i )); done; }
put_le32() { for i in 0 8 16 24; do put_byte $(( $1 >> i )); done; }
```

Both of these are pretty ugly ways to round-trip integers to/from binary. The problem is that we don't have any primitives for this operation in particular, so we are having to use the standard POSIX tooling to construct what we want manually, and that tooling is rather hostile to this operation because it assumes that we are trying to operate on text, and not bytes. The POSIX `printf` trick - a leading single quote on an argument yields the first char's byte value - works, but it feels like a hack.

Ultimately this resulted in two new primitive builtins being written:
* `readint` - reads a binary integer (up to 128 bits) from stdin and prints its decimal value.
* `writeint` - takes an integer (argument or stdin, decimal or `0x`/`0b`/`0` notation), and prints its byte representation at a fixed width (`-w <bits>`)

These additions alone made these helpers completely obsolete, and the main loops transform like this:

```sh
# parse_fmt loop
while thru -T 52 -v chunk <&5; do
    [ "$(len -b "$chunk")" -eq 52 ] || break
    flog INFO "parsing chunk #$((++i))"

    # write the full chunk to this brace group
    printf '%s' "$chunk" | {
        # thru -T consumes set portions of it
        thru -T 16 | thru --until $'\0' | push data # name
        readint -w 32                   | push data # offset
        thru    -T 8                    > /dev/null # unknown
        readint -w 32                   | push data # length

        # wav file headers
        for chunk in 2 2 4 4 2 2 2; do
            readint -w $(( chunk * 8 )) | push data
        done

        thru -T 2 > /dev/null # padding

        if thru -v leftover; then
            local remainder=$(len -b "$leftover")
            flog WARN "leftover data after chunk #$i: $remainder bytes"
        fi
    }

    echo "${data[@]}"
    data=()
done

# write_wav loop
while read -r name off size tag ch rate avg align bits cbsize; do
    {
        flog INFO "writing $name"

        printf "RIFF"
        writeint -w 32 "$(( 36 + size ))"
        printf "WAVE"

        printf "fmt "
        writeint -w 32 "16"
        writeint -w 16 "$tag"
        writeint -w 16 "$ch"
        writeint -w 32 "$rate"
        writeint -w 32 "$avg"
        writeint -w 16 "$align"
        writeint -w 16 "$bits"

        printf 'data'
        writeint -w 32 "$size"

        # seek to '$off' bytes in the file
        seek 5 $off > /dev/null
        # read exactly '$size' bytes from the file
        thru -T $size <&5
    } > "$dir/$name" # write the file
    flog INFO "wrote '$dir/$name', $(printf '%h' "$size") bytes"
    total+=$size
done
```

Reads much more cleanly. Because `readint -w <width>` reads exactly its width off the stream and then stops, the fields don't need a `thru -T <bytes>` call, they handle the input themselves. The name field still needs two `thru` calls though; we need to consume all 16 bytes of the field, and then parse the actual name out of them. Just the `-T 16` call, and we end up with extra junk at the end. Just the `--until $'\0'` call, and we don't completely consume the field.

Every one of these additions came out of basically the same loop:
1. Find a spot where the script felt like it was fighting the shell
2. Realize `shed` is simply missing a specific primitive operation
3. Add the primitive as a builtin, or extend an existing builtin
Each time, whole stretches of the script would collapse into a single line.

I don't mean to overclaim off a single project, but doing this kind of work in a shell, cleanly (and quickly, 5.2GB extracted and written across all 13 games in under two seconds!), feels like a bit more than just a novelty. I'll wait to see it hold up on a few more problems before I make a real case for it, but I think there is a case to be made.
