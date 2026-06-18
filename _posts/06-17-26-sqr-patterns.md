---
layout: post
title: Shell Quoted Records
date: 2026-06-17
---
  This is the third post I've written in a series covering research I've been doing on a wire protocol that uses shell quoting to serialize fields and preserve structure across pipelines. The first post is [here](https://km-clay.github.io/2026/06/14/shell-structured-data.html), and the second post is [here](https://km-clay.github.io/2026/06/15/quoting-wire-protocol.html), if you wish to read those first.

  I've taken to calling this protocol *"Shell Quoted Records"*, or SQR for short, as a citation convenience. Anyway, I've been doing some more work on finding practical uses for SQR in common shell scripting contexts. In the last post, I detailed a bunch of `nushell`-like combinators that allowed for filtering, re-arranging, or mapping functions to records. I also showed off how CSV can be translated into SQR and then run through these combinators.

  I was looking for some more inspiration, so I decided to take a closer look at how `nushell` actually handles stuff like this. `nushell`'s records are a data structure that's internal to the shell, and the combinators that they can be filtered through are also builtins. When the records actually exit a pipeline, they are then formatted into a pretty table and written to stdout. That's my impression of how it works, anyway. There are a few differences between `nushell`'s structured records and `shed`'s SQR:
  1. `nushell`'s records are internal to the shell, which makes interop with external utilities less practical. SQR is a flat byte stream that follows a convention, so it is fully compatible with external tools. However, working with the data feels a little bit more stilted, as with most things in POSIX shell scripting.
  2. `nushell`'s records are formatted into a pretty table upon leaving the shell, while SQR is written to stdout literally. We don't get formatting for free.
  3. `nushell`'s records have *named fields*. This is basically the biggest gap between the two systems, and is something I've been targeting for SQR.

Anyway, I decided to see if it would be possible to have `shed` use SQR to do two things that `nushell` does out of the box:
  1. `ls` that outputs structured data
  2. tables that render when SQR hits the terminal

  I started with the `ls` idea.
```bash
  qls() {

  # field extraction helper
  emit_fields() {
    quote "${fields[0]}" \
      "${fields[1]}" \
      "${fields[2]}" \
      "${fields[*]:3:3}" \
      "${fields[*]:6}"
  }
  defer unset -f emit_fields # cleanup

  local buf=""
  local first=1
  ls -l | while read -q -a fields; do
    if (( first )); then
      first=0
    else
      buf+=$'\n'
    fi

    buf+=$(emit_fields)
  done

  if [ -t 1 ]; then
    # stdout is a terminal
    # format the data into a table
    echo "$buf" | qtable # <- more on this later

  else
    # stdout is a pipe or something?
    # just pass it through
    echo "$buf"

  fi
}
```
  While working on this I figured out a nice pattern for packing tabular command output into SQR records. The `emit_fields` function defined toward the top there shows this pattern. The `$fields` array holds one line of the output of `ls -l`, which looks like this:
```bash
.rw-rw-rw- 0 pagedmov 17 Jun 16:54 file with spaces in it
---------- - -------- -- --- ----- ---- ---- ------ -- --
0          1 2        3  4   5     6    7    8      9 10
```
  With the lines and numbers under it just being a visual representation of what the shell sees as "fields" when this output is fed to `read -q`. Once these fields are packed into an array, the array is processed in that `emit_fields` function I mentioned earlier:
```bash
  emit_fields() {
    quote "${fields[0]}" \ # mode
      "${fields[1]}" \     # size
      "${fields[2]}" \     # user
      "${fields[*]:3:3}" \ # date ('day month HH:MM')
      "${fields[*]:6}"     # filename
  }
```
  Two very useful patterns emerge here:
  1. A range of fields can be merged together using slicing with a length
  2. The remaining fields can all be merged together using slicing with no length.

  Using `"${fields[*]:3:3}"`, we are able to capture the space-separated date parts into a single field.
  Using `"${fields[*]:6}"`, we are able to capture the 7th field and everything after it, which will catch any filename.

  The result of this is a correctly formatted capture of `ls`'s output as SQR:
```bash
$ qls
.rw-rw-rw- 0 pagedmov '17 Jun 16:54' 'file with spaces in it'
---------- - -------- -------------- ------------------------
0          1 2        3              4
```

  That's great and all, but it completely mangles the output visually. All visual formatting is lost in the process, and the actual emitted output looks like this:
```bash
$ qls
.rw-r--r-- 248 pagedmov '6 Jun 22:09' build.rs
.rw-rw-rw- 38k pagedmov '16 Jun 18:42' Cargo.lock
.rw-rw-rw- 1.4k pagedmov '16 Jun 18:42' Cargo.toml
.rw-rw-rw- 0 pagedmov '17 Jun 16:54' 'file with spaces in it'
.rw-r--r-- 2.0k pagedmov '5 Jun 18:32' flake.lock
.rw-r--r-- 3.0k pagedmov '16 Jun 18:42' flake.nix
.rw-r--r-- 285k pagedmov '6 Jun 17:20' flamegraph.svg
.rw-r--r-- 1.1k pagedmov '5 Jun 18:32' LICENSE
.rw------- 782M pagedmov '6 Jun 17:20' perf.data
.rw------- 760M pagedmov '6 Jun 17:20' perf.data.old
.rw-r--r-- 19k pagedmov '6 Jun 16:45' perf.report
.rw-r--r-- 18k pagedmov '7 Jun 22:35' README.md
.rw-r--r-- 72 pagedmov '5 Jun 18:32' rustfmt.toml
```
  Which is kind of gross to look at. Lucky for us though, we can do our own formatting since we have the data right here. I wanted to see how hard it would be to visually format our SQR output in the same way that `nushell` formats its tables. For the `qtable` implementation, I'm actually going to pack this one into a [gist](https://gist.github.com/km-clay/f1049a8888da1e0e58d3dc087d888a1b) because the function definition is >100 lines long.

  When combined with `qtable`, the output of `qls` looks like this:
```bash
$ qls
╭────────────┬──────┬──────────┬──────────────┬────────────────────────╮
│ drwxr-xr-x │    - │ pagedmov │  5 Jun 18:32 │                 crates │
│ drwxr-xr-x │    - │ pagedmov │  5 Jun 18:32 │               devtools │
│ drwxr-xr-x │    - │ pagedmov │ 16 Jun 04:13 │               examples │
│ drwxr-xr-x │    - │ pagedmov │  5 Jun 18:32 │                include │
│ drwxr-xr-x │    - │ pagedmov │ 11 Jun 21:34 │                    nix │
│ drwxr-xr-x │    - │ pagedmov │ 16 Jun 03:29 │                    ref │
│ drwxr-xr-x │    - │ pagedmov │ 16 Jun 18:04 │                release │
│ drwxr-xr-x │    - │ pagedmov │ 15 Jun 11:31 │                    src │
│ drwxr-xr-x │    - │ pagedmov │  5 Jun 19:39 │                 target │
│ .rw-r--r-- │ 1.7k │ pagedmov │  5 Jun 18:32 │           AI_POLICY.md │
│ .rw-r--r-- │  248 │ pagedmov │  6 Jun 22:09 │               build.rs │
│ .rw-rw-rw- │  38k │ pagedmov │ 16 Jun 18:42 │             Cargo.lock │
│ .rw-rw-rw- │ 1.4k │ pagedmov │ 16 Jun 18:42 │             Cargo.toml │
│ .rw-rw-rw- │    0 │ pagedmov │ 17 Jun 16:54 │ file with spaces in it │
│ .rw-r--r-- │ 2.0k │ pagedmov │  5 Jun 18:32 │             flake.lock │
│ .rw-r--r-- │ 3.0k │ pagedmov │ 16 Jun 18:42 │              flake.nix │
│ .rw-r--r-- │ 285k │ pagedmov │  6 Jun 17:20 │         flamegraph.svg │
│ .rw-r--r-- │ 1.1k │ pagedmov │  5 Jun 18:32 │                LICENSE │
│ .rw------- │ 782M │ pagedmov │  6 Jun 17:20 │              perf.data │
│ .rw------- │ 760M │ pagedmov │  6 Jun 17:20 │          perf.data.old │
│ .rw-r--r-- │  19k │ pagedmov │  6 Jun 16:45 │            perf.report │
│ .rw-r--r-- │  18k │ pagedmov │  7 Jun 22:35 │              README.md │
│ .rw-r--r-- │   72 │ pagedmov │  5 Jun 18:32 │           rustfmt.toml │
╰────────────┴──────┴──────────┴──────────────┴────────────────────────╯
```

  Not too shabby. Though there are some issues that will have to be ironed out. One issue is that of performance. `nushell` can do this, because all of the records and data manipulation are internal to the shell. For us, this is all being implemented as shell functions, which means we really need to watch our step to avoid forking as much as possible. I even implemented a `cat`-like builtin (`thru`) that literally just passes bytes directly from stdin to stdout, so that the `input=$(cat)` pattern could be internalized via `input=$(thru)`. Since `shed` doesn't use fork+exec for command subs that use builtins, this is much faster because `cat` shells out to an external program that *does* require a fork+exec to run.

  The other issue is that of actually *naming* the fields. `nushell`'s tables give names to each of the fields. This allows for very readable invocations like `ls | where size > 1000`. For us, naming the fields is going to be a little bit complicated. Since the SQR format is identical in spirit to CSV, we should probably just go with their solution: an optional convention that the first row contains the field names, with tools being told (or knowing in advance) whether the data has a header row or not.

  It's really more of a half solution than anything, and requires offloading some complexity to the users of your format, which I don't particularly like doing but there's not really a better alternative here. We can't add a row with some kind of sentinel markers, because only `shed` knows about those markers, so other tools like `sort` will corrupt the data. The CSV approach is what I've decided to go with for the first version of this protocol.

  That being said, I've added a `qname` combinator that takes SQR input from stdin and a list of names as its arguments, and then prepends the name list to the SQR data, so that it appears as the first record. Additionally, all of the combinators introduced in the last post have gained a `-n` flag that signals that their input has a header row, and each one handles the input accordingly:
  * `qmap`    -> Skips lambda for the header record, passing it through unchanged. Each field is mapped to a variable with a matching name in the lambda.
  * `qwhere`  -> Predicate doesn't run on the header. Fields also get mapped to a variable with their field's name.
  * `qselect` -> Accepts field names OR indices as arguments.
  * `qpad`    -> Skips the header record. Pads to the length of the header record instead of the longest record. Records longer than the header are not truncated.
  * `qcount`  -> Does not count the header record.
  * `qpipe`   -> Strip the header record, pipe the rest of the records into external, combine header record with transformed records
  * `qtable`  -> The field names are used as headers at the top of the table. Also renders under the table if it's too tall to fit in the terminal.

  With this change, working with named data will look something like this:
```bash
quote_csv | qname name email id priority msg \
  | qselect -n id name priority msg \
  | qpipe -n sort \
  | qwhere -n '[[ "$priority" == high ]]' \
  | qmap -n 'echo "[$id] - $name --- $msg"'
```
  The trade-off is having to pay a `-n` on every combinator that follows `qname`, in return for being able to address fields by name.

  If we go back to our `qls` function from earlier, we can now take this line:
```bash
  echo "$buf" | qtable
```
  And add `qname`:
```bash
  echo "$buf" | qname mode size user date name | qtable -n
```

  The result is a complete table reminiscent of `nushell`'s formatted output:
```bash
$ qls \
  | qwhere -n '[[ "$name" == perf* ]]' \
  | qselect -n name size user \
  | qtable -n
╭───────────────┬──────┬──────────╮
│          name │ size │     user │
├───────────────┼──────┼──────────┤
│     perf.data │ 782M │ pagedmov │
│ perf.data.old │ 760M │ pagedmov │
│   perf.report │  19k │ pagedmov │
╰───────────────┴──────┴──────────╯
```

  Its fascinating how many doors have been opened by the SQR format. I will continue researching this. Its clear that SQR has useful applications, but it hasn't been truly stress tested yet. I think the next thing on the list will be to see how it performs on extremely large data sets, and round-tripping through different formats.
