---
layout: post
title: chrono and fs
date: 2026-10-03
---

I just released `shed` v0.45.1 about an hour ago. The patch includes the new `fs` builtin, and some fixes/features for the `chrono` builtin. Both of these commands make strong use of the `BuiltinRouter` trait that I implemented a few weeks back to add support for builtins that have *subcommands*.

The `zd` builtin was the first in the codebase to have subcommands associated with it, and at the time that I implemented these subcommands, the logic was very simple and very hacky:

1. read the arguments
2. if the first argument matches a subcommand name, remove it from the arguments and delegate to a different function
3. otherwise, treat it as a search query

This pattern *works*, but it's not pretty. I couldn't really justify coming up with a proper framework though, since only one command in the entire codebase had subcommands, so once it worked I just kinda left it. Eventually though, after the addition of [command history branching](https://github.com/km-clay/shed/releases/tag/v0.43.0), `hist` also wanted subcommands. So now a framework was suddenly justified.

The `BuiltinRouter` trait effectively turns a builtin command into a simple dispatcher. Top-level dispatch matches words to structs that implement the `Builtin` trait, e.g. `keymap` -> `&Keymap`. Implementors of `BuiltinRouter` would have their own dispatch layer that does the exact same thing. So `hist` now receives the arguments, checks the next one, and then dispatches; e.g. `pull` -> `&HistPull`.

Writing a command with subcommands is now as easy as writing out something like this:

```rust

pub(super) struct Chrono;
impl BuiltinRouter for Chrono {
  fn default_sub(&self) -> &'static dyn Builtin {
    &ChronoError // prints usage and exits
  }

  fn sub_for(&self, word: &[u8]) -> Option<&'static dyn Builtin> {
    match word {
      b"timer" => Some(&timer::Timer      ),
      b"sleep" => Some(&sleep::Sleep      ),
      b"fmt"   => Some(&format::Format    ),
      b"every" => Some(&every::Every      ),
      b"zone"  => Some(&timezone::Timezone),
      _ => None,
    }
  }
}

impl Builtin for Chrono {
  // convert a &dyn Builtin to &dyn BuiltinRouter
  // used for nested BuiltinRouter implementors
  // e.g. "if this Builtin is a BuiltinRouter, then use the router interface instead"
  fn as_router(&self) -> Option<&dyn BuiltinRouter> {
    Some(self)
  }

  // delegate to BuiltinRouter trait methods
  fn get_argv_and_opts(&self, cmd_span: Span, argv: &[Tk], no_split: bool) -> ShResult<Parsed> {
    self.route_parse(cmd_span, argv, no_split)
  }
  fn execute(&self, args: super::BuiltinArgs) -> ShResult<()> {
    self.dispatch_sub(args)
  }
}
```

That's actually it. As long as the subcommands themselves are implemented, routing becomes trivial.

The design also allows for nesting and indirection of *arbitrary depth*, meaning that subcommands can *also* have subcommands. If a `Builtin` that matches a word is *also* a `BuiltinRouter`, it hands off dispatch to that router. This allows for stuff like `chrono timer`, which itself has five different subcommands for managing timer state. The dispatch loop continues until something is found that can actually be executed.

The `chrono` and `fs` builtins are the first to take full advantage of this design, and are essentially namespaces for groups of separate builtins. For chrono:

* `chrono timer` - manage and format named timers
* `chrono sleep` - pause for a given duration or until a given point in time
* `chrono fmt`   - convert time by applying a duration or instant to a format string
* `chrono every` - run a command repeatedly on an interval, with tracking of missed intervals and zero drift
* `chrono zone`  - print and filter portable timezone names

Having builtins that act as namespaces lets me name the commands very freely, since there's no chance of shadowing any existing commands. The tradeoff is of course, the fact that if the main builtin is itself shadowed, the entire namespace becomes inaccessible. I think it's a worthy trade.

As for the commands themselves, `chrono` and `fs` have been two pretty big wins in my opinion.

## The `chrono` builtin

`chrono` provides a rather powerful interface for generally working with time in scripts. The subcommands operate on "durations" (spans of time) or "instants" (exact moments in time), and have uniquely capable parsing for both forms:

* instants - `2026-12-25 09:00:00`, `4:00 pm`, `2 hours from now`, `5 days ago`, `@1345349809` (epoch seconds)
* durations - `5 hours`, `1h30m`, `october 5 2024 to october 10 2024` (span of time between two or more instants)

The parsers are even capable of handling deeply nested expressions as well, for instance:

```sh
#                |------------------------------------------------duration-----------------------------------------------|
#                v                                                                                                       v
$ chrono fmt -d '3 days before 2 hours after 9:00pm 10/13/26 to @1700000000 to 1 week after 2 min before 5:00am 2026.12.25'
#                ^             ^             ^             ^    ^         ^    ^            ^            ^               ^
#                |             |             |---instant---|    |-instant-|    |            |            |-----instant---|
#                |             |---------instant-----------|                   |            |----------instant-----------|
#                |-----------------instant-----------------|                   |-----------------instant-----------------|

2 months 3 weeks 1 day 6h 58m
```

The subcommands themselves enable interaction with time that's usually very clunky in existing shells. `bash` and `zsh` provide `$EPOCHREALTIME`/`$EPOCHSECONDS` for crude time calculation using "seconds since the Unix epoch" as an anchor. I have not been able to find anything like `chrono every` (drift-proof recurring command scheduler) built into any shell that I've looked through, closest I've seen is `zsh/sched` which is strictly-oneshot. `chrono sleep` provides all of the sane uses of GNU `sleep`, and also provides a way to sleep *until* a specific point in time, instead of for a set, prescribed duration: `chrono sleep until "5pm tomorrow"`

I've already personally used `chrono timer` for benchmarking scripts, and `chrono every` is currently managing my shell command history backups. `chrono sleep`, on top of the extra capabilities, is capable of being *far more precise* (accurate on the scale of microseconds) since fork overhead is a non-issue.

Speaking of my daily history backups, that script is actually pretty interesting:

```sh
backup_daemon() {
    local last_run= size=
    local STATE=~/.local/state/shed
    local state_file=$STATE/last_backup
    local backup_file=$STATE/hist_backup.json

    exec {lockfd}<> $STATE/backup.lock
    defer exec {lockfd}>&-

    lock -n $lockfd || return
    defer lock -u $lockfd # LIFO - unlock file, then close fd


    try {
        last_run=$(thru "$state_file") # read the state file
        chrono fmt -f "%x" "$last_run" > /dev/null # validate contents
    } catch; do {
        # write to both the state file and this last_run var
        last_run=$(chrono fmt -f "%x" now | thru -t "$state_file")
    } done

    # if this function is already set, save the definition
    local old=$(declare -f run_backup)
    # restore it if it did, or unset it if it didnt
    defer eval "${old:-unset -f run_backup}"

    run_backup() {
        hist export            > "$backup_file"
        chrono fmt -f "%x" now > "$state_file"
        size=$(stat "$backup_file" -c '%S')

        # function from my rc; writes a status message to
        # every currently open-and-writable `shed` IPC socket
        shout "shell history backed up [$size]"
    }

    chrono every "24 hours" "run_backup" \
        --queue 1 \
        --catch-up \
        --starting "$last_run"
}
```

There's a few things going on here. The first is the use of the `lock` builtin, which is a pretty recent addition. Successfully locking that file means that this is the only backup daemon running. If the lock can't be acquired, the script exits. The `try` block that follows basically does the following:

1. if we can read the state file, store the date it contains and continue,
2. if we can't read it, then format 'now' as "%x" to get the current date, then use `thru -t "$state_file"` to write it to the state file while also emitting output for the command substitution it's inside of.

After we've gotten the date for the last time a backup occurred, we set this `run_backup` function to be used as the handler executed by `chrono every`. This is one of the strengths of having a scheduler built into the shell itself - it can access and execute shell functions directly. While I was writing out this script, I found an interesting pattern. Say you're in a function already, and you'd like to write some throwaway helper you don't need anywhere else.

```sh
some_func() {
# ...
    echo foo

    some_helper() {
        echo biz
    }


    echo bar
# ...
}
```

The problem with this is that now you've polluted the global namespace. `some_helper` is a function that can be seen by everyone now. I did find a way around this though:

```sh
some_func() {
# ...
    echo foo

    local old=$(declare -f some_helper)      # store function definition
    defer eval "${old:-unset -f some_helper}" # defer restoration or unset
    some_helper() {
        echo biz
    }


    echo bar
# ...
}
```

Now, no matter what, the moment you leave this function, everything will be left just the way you found it. How considerate!

Anyway, we're now at the main event of this script:

```sh
    chrono every "24 hours" "run_backup" \
        --queue 1 \
        --catch-up \
        --starting "$last_run"
```

It's a compact command, but this is actually expressing quite a bit. Let's walk through the arguments. First: `every "24 hours" "run_backup"` - this one's easy, it basically reads like plain english. Every 24 hours, execute "run_backup". Next is `--queue 1` which is a little bit less obvious.

Basically, `--queue {n}` lets you queue up "missed executions". If you've passed at least one interval since the command's "anchor instant" (either the moment it started, or the instant given to `--starting`), then you have "missed" an execution. The number of intervals that have passed is the number of missed executions, and the number to actually execute is capped by the `--queue` argument. `--catch-up` tells the command to count the number of missed intervals relative to the instant given to `--starting`. Normally, `--starting` is only used for interval alignment, but `--catch-up` also lets you use it to count how many executions have been missed since some moment in the past. This allows the builtin to *also be aware of time spent not running*, so keeping track of owed executions can even be done between reboots!

## The `fs` builtin

`fs` is somewhat of a different beast though. The main idea is pretty well-trodden ground: give the shell a toolkit for direct filesystem operations. The POSIX language only really exposes the barest essentials for operating on files; mainly just redirection and `exec` with file descriptors. Real filesystem operations are almost always outsourced to external programs. The `fs` set, and stuff like the builtins `zsh` provides with the `zsh/files` module, seek to remedy this issue by giving the shell itself the capability to do filesystem operations independently. The `fs` builtin exposes these syscalls as subcommands:


| subcommand | system call | behavior |
|------------|-------------|----------|
| `fs rename <from> <to>` | `rename(2)` | renames the target file/directory/link. fails across filesystems, unlike `mv` |
| `fs rmdir <dir> ...` | `rmdir(2)` | removes each `dir`, if the directory is empty. |
| `fs unlink <file> ...` | `unlink(2)` | remove files. if a file ends up with no links after this, the data is deleted. |
| `fs link <from> <dest> ...` | `link(2)` | create hard links to a file. a file's data will survive as long as one hard link remains. |
| `fs symlink <from> <dest> ...` | `symlink(2)` | create symbolic links to a file. has no bearing on target file's lifetime |
| `fs readlink <link>` | `readlink(2)` | print the target stored in `link`. only resolves one layer of nested links. |
| `fs realpath <file>` | `realpath(3)` | print the canon representation of a given path. follows all layers of nested links. |
| `fs truncate <size> <file> ...` | `truncate(2)` | set each `file` to `size` bytes. does not create missing files |
| `fs chown [-h] <owner> <file> ...` | `chown(2)`/`lchown(2)` with `-h` | set the owner/group of each `file` |
| `fs chmod <mode> <file> ...` | `chmod(2)` | set the permissions of each `file`, `mode` may be octal or symbolic |
| `fs mkdir [-m <mode>] <dir> ...` | `mkdir(2)` | create each `dir`, optionally specifying a mode with `-m <mode>` |
| `fs touch [-amh] [-t <time>] <file> ...` | `utimensat(2)` | set the access and modify times of each file. does not create missing files. |

Each subcommand (except for realpath) is basically just a transparent passthrough to a corresponding system call. The logic of each command is dictated by the logic of the system call it wraps, and they make zero assumptions about the caller's intentions.

You may have noticed that these mirror a lot of the coreutil commands. The coreutils that share names with our `fs` commands are incidentally also thin wrappers around a system call. Of course, they ship with a lot more options, but the bare invocation is still 80% of the use cases, apart from maybe `mkdir -p`.

Speaking of `mkdir -p`; why didn't we implement that? We have `fs mkdir` after all. Well, I kind of decided while implementing these that the commands should basically just be "dumb tools". These are meant to be primitives, not programs. `fs mkdir` is the substrate on which things like `mkdir -p` can themselves be implemented:

```sh
mkdirall() {
    for target in "$@"; do
        local segments
        local path=""

        split '/' "$target" | unquote -a segments

        for segment in "${segments[@]}"; do
            # empty segments come from a leading slash
            [ -z "$segment" ] && { [ -z "$path" ] && path="/"; continue; }

            case "$path" in
                ""|*/) path+="$segment" ;; # empty or relative
                *)     path+="/$segment";; # absolute
            esac

            [[ -d "$path" ]] || fs mkdir "$path" || return 1
        done
    done
}
```

And this of course is composed entirely out of `shed` builtins. This is just an example, if you really needed `mkdir -p` for something, the coreutil is always right there waiting for you. This sort of primitive composability is what I was going for with these subcommands, and is the reason why stuff like `cp` and `find` did not make it into the `fs` namespace.

The commands that *did* make it into the `fs` set here are all *irreducible operations* - "atoms" of the domain, if you will. Their logic cannot be expressed by the shell's language; everything that they actually do is hidden behind the operating system interface. Stuff like `cp` and `find` on the other hand are themselves made up of these "atoms", and are therefore reducible. If it *can* be reduced, then that means **the shell can already express it.** `cp` is literally just `thru "$from" > "$to"` + `fs chmod`. `find` is a simple directory traversal with a pattern to filter by.

As an example, here's a ~40 line implementation of a `cp`-like tool that chunks files and writes to the target concurrently using background jobs as "worker threads":

```sh
ccp() {
    # "concurrent copy"
    local from="$1"
    local dest="$2"
    local nprocs="${3:-4}"

    [ -f "$from" ] || raise "'%(1)': no such file or directory" "$from"
    > "$dest"      || raise "'%(1)': failed to open destination for writing" "$dest"

    local size=$(stat -c '%s' "$from")

    # less than 8 mb, just call thru on it
    if (( size < (8 * 1024 * 1024) )); then
        thru "$from" > "$dest"
        return
    fi

    # resize destination file
    fs truncate "$size" "$dest"

    # derive chunk size
    # 2GB base ceiling'd against filesystem block size
    local bs=$(stat -f "$dest" -c '%S')
    local chunk=$(( 2 * 1024 * 1024 * 1024 ))
    chunk=$(( ((chunk + bs - 1) / bs) * bs ))

    local i workers
    let i=0;
    let workers=0;
    while (( i < size )); do
        { # forks here because of the exec call
            exec {dest_fd}<> "$dest"

            # seek to the start of the chunk in the target file
            seek $dest_fd $i > /dev/null

            # skip to the start of the chunk in the source file
            # take to the end of the chunk in the source file
            # write resulting bytes to the target file
            thru "$from" \
                --skip $i \
                --take $chunk \
                >&$dest_fd
        } & # <- background

        # when we hit {nprocs} number of workers
        # we stop and wait for all of them to finish
        # basically sending workers in waves
        i=$(( i + chunk ))
        workers=$(( workers + 1 ))
        (( workers >= nprocs )) && { wait; workers=0; }
    done
    wait

    echo "ok"
}
```

Note that this entire function is composed entirely of `shed` builtins.

It can just do this, out of the box, with ***zero dependence on the environment***. The only requirement for running this script is having the `shed` binary on your computer. This has many implications, but perhaps the most important is *portability*. Take a look at this line from the function:

```sh
local bs=$(stat -f "$dest" -c '%S')
```

On GNU coreutils, `stat -f` means `--file-system`. On BSD and macOS, `stat -f` is the *format string* flag. The same flag means two completely different things, and GNU's `-c` doesn't even exist there at all. This is not a problem we will ever have to deal with though; if you are executing this script, `stat` will only ever mean one thing - the builtin called `stat` that is embedded in the binary of the program itself.

There's also the case of truly minimal environments: distroless containers, initramfs images with no coreutils at all, and recovery situations where `/usr` isn't mounted or `PATH` is broken but a static shell still runs. If you ever find yourself disconnected from your externals, you will always have something to fall back on.

I decided to place all of these subcommands under a single `fs` namespace, because implementing these as direct standalone builtins would shadow the existing coreutils, and replacement is not really the idea here. There are some builtins where shadowing is fine, such as `stat` and `printf`, I usually take stuff like this in the help text as a green light:

```
Your shell may have its own version of printf, which usually supersedes
the version described here.  Please refer to your shell's documentation
for details about the options it supports.
```

`stat` has the same blurb. Notably, the coreutils like `cp` and `mv` *don't* have this blurb. This isn't a hard and fast rule, but it does inform my decision.

Initial profiling for these has shown to be very promising for certain big-batch file operations. When compared to a corresponding coreutil that takes all filenames up front, the `fs` builtins were only a *little bit* faster. However, for operations that worked on *one file at a time* (e.g. batch renaming), `fs` completely smoked the coreutils.

```
$ shed ./ref/bench/fs_bench.shed 10000
N = 10000 files, release build, timed with chrono timer

A. batched (one invocation, N operands)
╭────────────────────────────────┬────────┬───────────────╮
│ command                        │ micros │ micros_per_op │
├────────────────────────────────┼────────┼───────────────┤
│ fs truncate 0 "$W"/w/*         │ 28802  │ 2             │
│ command truncate -s 0 "$W"/w/* │ 38669  │ 3             │
╰────────────────────────────────┴────────┴───────────────╯

B. per-file loop
╭────────────────────────────────────────────────────────┬─────────┬───────────────╮
│ command                                                │ micros  │ micros_per_op │
├────────────────────────────────────────────────────────┼─────────┼───────────────┤
│ for f in "$W"/w/*; do fs truncate 0 "$f"; done         │ 40002   │ 4             │
│ for f in "$W"/w/*; do command truncate -s 0 "$f"; done │ 9312560 │ 931           │
╰────────────────────────────────────────────────────────┴─────────┴───────────────╯

C. unbatchable (rename each to $f.bak)
╭──────────────────────────────────────────────────────┬─────────┬───────────────╮
│ command                                              │ micros  │ micros_per_op │
├──────────────────────────────────────────────────────┼─────────┼───────────────┤
│ for f in "$W"/w/*; do fs rename "$f" "$f.bak"; done  │ 78459   │ 7             │
│ for f in "$W"/w/*; do command mv "$f" "$f.bak"; done │ 9969773 │ 996           │
╰──────────────────────────────────────────────────────┴─────────┴───────────────╯

D. shell loop overhead alone (no filesystem work)
╭───────────────────────────────┬────────┬───────────────╮
│ command                       │ micros │ micros_per_op │
├───────────────────────────────┼────────┼───────────────┤
│ for f in "$W"/w/*; do :; done │ 8328   │ 0             │
╰───────────────────────────────┴────────┴───────────────╯

ratios:
A: 1x   B: 232x     C: 127x
```

Which is kind of to be expected since the real bottleneck is fork overhead. The script used to run this benchmark can be found [here](https://gist.github.com/km-clay/75df463161c4c890d5243f94abffe078) (it uses `chrono` for the benchmarks!).

As I've stated in some of the more recent blog posts, my main idea for `shed`'s design at this point is to create a shell that acts more like a general mediator between the user and the operating system, rather than being a simple program launcher/script runner. Therefore, it stands to reason that the shell itself should expose the primitives of the operating system in its interface, and this of course includes the filesystem syscalls. `chrono` falls under this umbrella as well, it just exposes some rather obscure aspects of Unix, namely stuff like the combination of `clock_nanosleep` + `TIMER_ABSTIME` + `CLOCK_MONOTONIC` being able to align a command scheduler to a specific interval.

There is, naturally, a tradeoff that comes with having an interface with a surface this wide; `shed` as of right now has 95 builtin commands and 33 subcommands, so a total of **128** builtin commands if we're counting all implementors of the `Builtin` trait. That is a significant maintenance burden. I'd like to think that the framework I designed will continue holding up as the shell is continuously extended, but as with all things in software engineering, you just never really know.

There's also the fact that `shed` is still a young program and several of its builtins are actual novelties, like `vice`, or `forget` for instance. In such circumstances, bugs could be hiding really anywhere, and testing these builtins is not as straightforward as testing something like `printf`, that has actual decades worth of discovered footguns and extremely detailed documentation.

Overall, the past few patches have improved `shed`'s independence quite significantly. The program is actually (fairly swiftly) approaching a state where it could reasonably bootstrap its own userland. The scripting language is pretty much complete at this point in my opinion, I think all that's really left for now is just exposing more of the OS' interface. There is a limit to how far we can reasonably go with this since `shed` has no way to represent pointers, but I feel like the hard limit is still pretty far away.
