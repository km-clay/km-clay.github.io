---
layout: post
title: Salvaging vicut
date: 2026-06-19
---

  `vicut` is a neat program that I wrote a little over a year ago now. The idea of the program was to allow for slicing input text using `vim` motions. Since `vim`'s motions are so aware of structure, it would be useful to be able to use those motions to extract fields from command output. That was the original idea anyway, the project's scope eventually ballooned in size and I kind of gave up on it.

  By the time I stopped working on it, the feature set was far beyond the initial idea:
  * `--template` lets you provide a format string to interpolate the extracted fields into
  * `--delimiter` lets you give some pattern to put between the fields
  * `--json` emits the fields as json
  * `--linewise` performs the vim commands you give it on each line instead of the whole buffer
  * `--trim-fields`, `--trace`, `--silent`, `--serial`, `--global-uses-line-numbers`(???), the list goes on.
  * a literal entire (buggy) JS-like scripting language

  It was getting to be a bit much. The idea for the program came to me while I was working on `shed`, my unix shell, about a year ago. As I was scaffolding the module that would eventually become the `shed` line editor, I realized it would be pretty cool to have some way to script the line editor in some way. I was working with some kind of tabular command output, and thought how nice it would be if I could just use `vim` to slice fields directly since some command output I was working with wasn't cooperating with standard tooling because of some idiotic use of whitespace in the output preventing `cut` from reading it correctly.

  Anyway, I remembered about this program earlier today while I was working on `shed`, and it occurred to me that `vicut` could actually fit in very cleanly as a `shed` builtin. `vicut` itself used `shed`'s editing engine internally, so exposing `vicut` as a builtin would basically just mean having a command that calls the line editor logic with prescribed editing commands. I wouldn't need to extract the line editor as some crate that two projects share, both commands could draw from the exact same source.

  The issue with this is the fact that `vicut` has a lot of standalone value, but becomes intrinsically tied to `shed` as a builtin utility. This isn't a problem for me personally, since I am technically writing `shed` specifically for my own personal use, but I can see how that might be disappointing to someone who really likes the idea of `vicut` but doesn't want to deal with a wholesale move to another shell. Either way, I decided that I personally find value in `vicut`'s design, but I don't care enough to write it as a standalone project. So, `shed` builtin it is.

  First order of business when implementing this as a builtin: *picking a different name*. `vicut` sounded cool at first, but saying it is a chore. How do you even say it anyway? "vee eye cut"? "vy cut"? No possible pronunciations really roll off the tongue. I started regretting the name choice a while after starting the project, but by the time I cared enough to think about changing it, someone had already packaged it. Not the case here, so we're going to rename it `vice` - "vi" and "slice" put together.

  The actual implementation of the shell builtin turned out to be shockingly simple. Since `shed`'s line editor is already heavily featured and tested ***extensively***, the driving logic is already basically done. The internal getopts system for builtins handles sorting arguments and options. We already have a two-way parser/expander for `vim` keymaps. Essentially every component of the `vice` builtin was complete before I even started writing it. The implementation of `vice` is 312 lines of code. For perspective, `vicut`'s codebase is *17789* lines of code. Of course, `vice` itself is sitting on top of a ***74746*** line codebase, but who's counting anyway?

  The logic turned out to be much closer to `vim` than it was in the original implementation, since the editor is in a much more mature state. As a demonstration of what it can do, here's `vice` extracting the definition of `shed`'s `main()` function:

```bash
$ vice src/main.rs -m '/fn main<CR>0' -c '$%'
fn main() -> ExitCode {
  let Some(args) = lifecycle::setup() else {
    return ExitCode::SUCCESS;
  };

  if let Err(e) = input::dispatch_input(args) {
    if let ShErrKind::CleanExit(code) = e.kind() {
      QUIT_CODE.store(*code, Ordering::SeqCst);
    } else {
      e.print_error();
      if QUIT_CODE.load(Ordering::SeqCst) == 0 {
        QUIT_CODE.store(1, Ordering::SeqCst);
      }
    }
  }

  lifecycle::tear_down()
}
```
  It's very useful for finding some pattern in text, and then using the structural awareness of `vim` motions to extract the entire thing. `%` is a particularly powerful motion for this, since it completely handles arbitrary nesting of braces. On top of just reading and outputting text, it can of course perform edits on the text that it finds. For instance, here we extract it, and replace `QUIT_CODE` with `EXIT_CODE`:
```bash
$ vice src/main.rs -m ':%s/QUIT_CODE/EXIT_CODE/' -m '/fn main<CR>0' -c '$%'
fn main() -> ExitCode {
  let Some(args) = lifecycle::setup() else {
    return ExitCode::SUCCESS;
  };

  if let Err(e) = input::dispatch_input(args) {
    if let ShErrKind::CleanExit(code) = e.kind() {
      EXIT_CODE.store(*code, Ordering::SeqCst);
    } else {
      e.print_error();
      if EXIT_CODE.load(Ordering::SeqCst) == 0 {
        EXIT_CODE.store(1, Ordering::SeqCst);
      }
    }
  }

  lifecycle::tear_down()
}
```
  And the `-i` flag allows us to apply the edits made with `vice` back to the original file. This makes `vice` a very strong choice for performing bulk file edits that need an editor with structural awareness.

  For the input-stream side of things, take a look at this line of `ls` output:
```bash
-rw-r--r-- 1 pagedmov users 547 Jun 19 02:15 file with spaces in it
---------- - -------- ----- --- --- -- ----- ---- ---- ------ -- --
1          2 3        4     5   6   7  8     9    10   11     12 13
```
  Annotated with the way simple tools like `cut` would see this output. You may recognize this format from the last post I made about SQR. More on that later, for now, how can we use `vice` to group this output into actual fields? Like this:
```bash
  $ command ls -l 'file with spaces in it' | vice -d ' /// ' --sep 'W' \
2 |      -c 'viW' \
3 |      -m 'W' \
4 |      -r 2:1 \
5 |      -c 'viW' \
6 |      -c 'viWEE' \
7 |      -c '$'
-rw-r--r-- /// pagedmov /// 547 /// Jun 19 02:15 /// file with spaces in it
----------     --------     ---     ------------     ----------------------
1              2            3       4                5
```
  Very cool. So how did it do that? Each `-c <stuff>` call cuts a new field. It takes the span of text between where the cursor started and ended, and slices it out of the buffer as a field. The `-c 'viW'` and `-c 'viWEE'` commands use visual mode with text objects to carve out specific words. `--sep` defines a motion that occurs after each `-c` call, to set up the next cut. In this case the `--sep` is 'W', which moves to the start of the next word. `-m 'W'` is used, combined with the `--sep` motion, to skip over the second field since we want to land on the user field there. `-r 2:1` repeats the last 2 commands 1 time. `-d ' /// '` is used to define the field delimiter for the final output.

  And that's basically it. Well, I guess there is one more thing. Remember that SQR post I mentioned earlier? Surprise, this is actually another post about SQR in disguise. Check this out:
```bash
  $ command ls -l 'file with spaces in it' | vice --quoted --sep 'W' \
2 |      -c 'viW' \
3 |      -m 'W' \
4 |      -r 2:1 \
5 |      -c 'viW' \
6 |      -c 'viWEE' \
7 |      -c '$'
-rw-r--r-- pagedmov 547 'Jun 19 02:15' 'file with spaces in it'
---------- -------- --- -------------  ------------------------
1          2        3   4              5

```
  The `-q`/`--quoted` flags emit the fields in SQR format instead of using a delimiter. This can then be used as a producer for SQR consumers like `qtable`. In the last SQR post, I demonstrated an `eza` wrapper that used SQR to make a `nushell`-like table from the output. The `qls` function implementation I landed on was this:
```bash
qls() {
	emit_fields() {
		quote "${fields[0]}" \
		"${fields[1]}" \
		"${fields[2]}" \
		"${fields[*]:3:3}" \
		"${fields[*]:6}"
	}
	defer unset -f emit_fields

	local buf=""
	local first=0
	command eza -l --color=always | while read -q -a fields; do
		if (( first == 0 )); then
			buf+=$(emit_fields)
			first=1
			else
			buf+=$'\n'
			buf+=$(emit_fields)
		fi
	done

	local SQR_TABLE_JUSTIFY="left"
	echo "$buf" \
		| qname mode size user date name \
		| qselect -n name size user date mode \
		| __emit_sqr -n
}
```
  It works, but it is somewhat verbose. And the `"${fields}"` splitting is clever, but it has issues. For instance, it mangles whitespace between the fields. `some     file             with spaces` is crunched down to `some file with spaces`, due to the way that `[*]` joins strings. However, with the introduction of `vice`, this function becomes simpler, more logically accurate, *and way faster*:
```bash
qls() {
	local SQR_TABLE_JUSTIFY="left"

	command ls -l | tail -n +2 \
		| vice --lines --quoted --sep 'W' \
			-c 'viW' \
			-m 'W' \
			-r 2:1 \
			-c 'viW' \
			-c 'viWEE' \
			-c '$' \
		| qname mode user size date name \
		| qselect -n name size user date mode \
		| __emit_sqr -n
}
```
  It also uses GNU `ls` now for portability. The `--lines` flag makes `vice` execute the given editor commands per-line instead of once on the entire buffer, which is a perfect fit for translating tabular data into SQR. Not only does it simplify this function, it's also a hell of a lot faster. Instead of doing a `read` loop over the lines we get from `ls`, `vice` performs its own internal loop over those lines. This, coupled with a recent optimization that saves forks in builtin-heavy pipelines, causes the runtime to drop from *75ms* to render the table, to just *13ms*:
```bash
$ time qls
╭────────────────────────┬───────────┬──────────┬──────────────┬────────────╮
│ name                   │ size      │ user     │ date         │ mode       │
├────────────────────────┼───────────┼──────────┼──────────────┼────────────┤
│ AI_POLICY.md           │ 1728      │ pagedmov │ Jun  5 18:32 │ -rw-r--r-- │
│ build.rs               │ 248       │ pagedmov │ Jun  6 22:09 │ -rw-r--r-- │
│ Cargo.lock             │ 37832     │ pagedmov │ Jun 18 01:46 │ -rw-rw-rw- │
│ Cargo.toml             │ 1418      │ pagedmov │ Jun 18 01:46 │ -rw-rw-rw- │
│ crates                 │ 4096      │ pagedmov │ Jun  5 18:32 │ drwxr-xr-x │
│ devtools               │ 4096      │ pagedmov │ Jun  5 18:32 │ drwxr-xr-x │
│ examples               │ 4096      │ pagedmov │ Jun 18 02:02 │ drwxr-xr-x │
│ file with spaces in it │ 0         │ pagedmov │ Jun 19 02:15 │ -rw-r--r-- │
│ flake.lock             │ 2038      │ pagedmov │ Jun  5 18:32 │ -rw-r--r-- │
│ flake.nix              │ 3038      │ pagedmov │ Jun 18 01:46 │ -rw-r--r-- │
│ flamegraph.svg         │ 285467    │ pagedmov │ Jun  6 17:20 │ -rw-r--r-- │
│ include                │ 4096      │ pagedmov │ Jun  5 18:32 │ drwxr-xr-x │
│ LICENSE                │ 1067      │ pagedmov │ Jun  5 18:32 │ -rw-r--r-- │
│ nix                    │ 4096      │ pagedmov │ Jun 11 21:34 │ drwxr-xr-x │
│ perf.data              │ 782249656 │ pagedmov │ Jun  6 17:20 │ -rw------- │
│ perf.data.old          │ 759952384 │ pagedmov │ Jun  6 17:20 │ -rw------- │
│ perf.report            │ 19162     │ pagedmov │ Jun  6 16:45 │ -rw-r--r-- │
│ README.md              │ 17592     │ pagedmov │ Jun  7 22:35 │ -rw-r--r-- │
│ ref                    │ 4096      │ pagedmov │ Jun 16 03:29 │ drwxr-xr-x │
│ release                │ 4096      │ pagedmov │ Jun 18 01:43 │ drwxr-xr-x │
│ rustfmt.toml           │ 72        │ pagedmov │ Jun  5 18:32 │ -rw-r--r-- │
│ src                    │ 4096      │ pagedmov │ Jun 17 20:15 │ drwxr-xr-x │
│ target                 │ 4096      │ pagedmov │ Jun  5 19:39 │ drwxr-xr-x │
╰────────────────────────┴───────────┴──────────┴──────────────┴────────────╯

real	0m0.013
user	0m0.010
sys	  0m0.006
```

  I'm pretty happy with how cleanly it has landed in the codebase, though I still have somewhat mixed feelings about the inclusion. Including `vice` as a builtin is ultimately a statement about `shed`'s identity as a program. Traditionally, shells have taken a much more hands-off approach when it comes to functionality, only implementing what is necessary to manage processes, run scripts, and handle I/O. Lately, `shed` has seen a lot of additions that could be reasonably seen as superfluous or "overreaching". Like, `thru` is literally just `cat` and `tee` but missing some of their flags.

  I can see how someone who is used to using unix tools might find this strange. "Why not just use `cat` or `tee`?" is a pretty reasonable question. It's my honest opinion though, that the shell *should* be able to natively do stuff as fundamental as what programs like `cat` and `tee` do. The tradeoffs are clear: Either build in some very basic functionality (`thru` is ~80 lines), or pay the `fork()` + `execve()` tax any time you want to access that functionality, and you already saw just above what saving forks can do for performance. Shells have been doing this kind of thing for a while anyway; `printf` and `echo` are prime examples. `shed` just takes it a bit farther.

  On the other hand, `vice`'s functionality is more specialized than fundamental, so it can't really fall in the same bucket as `thru`. However, it is effectively just exposing an existing subsystem (the text editing engine) from a different angle. This is the same for a few other `shed` specific builtins, like `msg` and `flog` interacting with the system message queue, and `raise` allowing you to use the internal error propagation logic.

  Overall, going forward I think a builtin earns its place if it:
  1. exposes an API for existing subsystems/logic cheaply (`vice`, `msg`, etc)
  2. commonly lands in pipelines/loops where forking is very costly (`printf`, `thru`, etc)

  This may result in having a very wide library of builtins, but a program called "shed" feels like it should have a lot of tools in it anyway.
