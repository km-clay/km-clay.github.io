---
layout: post
title: Writing a Fast Unix Shell
date: 2026-06-14
---

  `shed`, my [Unix shell](https://github.com/km-clay/shed), is now reaching the point where I'm genuinely running out of ideas for things to add. This program has been in the oven for nearly 3 years now, and has suffered no less than 13 ground-up rewrites while I wrapped my head around how the hell a computer is supposed to read and execute something as awkward and stilted as the POSIX shell language (just look at the horror that is the [here-document specification](https://pubs.opengroup.org/onlinepubs/9799919799/)). Sometime last year though, I landed on an architecture that actually stuck, and from there just kept writing.

  It's a little bit weird having a project actually reach the point where optimization doesn't feel premature. Most of my projects up until this point have been one-off tools that only I have a use for, or just random curiosities like [whoa](https://github.com/km-clay/whoa), none of which have ever reached the scale or level of completeness required for optimization/profiling to really matter. `shed` has turned out differently though. The shell is my main method of interacting with my computer, and having my own shell and daily driving it means I have plenty of incentive to basically never abandon the project, and make it as genuinely nice to use as I can. The current feature set seems relatively complete (in my opinion), so now all that's left is just making it fast.

  To actually profile `shed` and find places where it might be lacking in terms of speed, I chose to use `hyperfine` along with some stressful home-made scripts, comparing `shed`'s execution time to `bash`, `zsh`, and `fish`. Final results of these benchmarks as of today can be found at the bottom of this post.

  A problem I found pretty much immediately was something that I imagine most people run into when they reach this phase of a project, especially in Rust; clones and allocations literally everywhere. For one, `shed`'s state management system involves interacting with global variables through closures. Normal borrows cannot be pulled out of these closures, since the underlying `RefCell` borrow is dropped when the closure returns. Values can be easily moved down into these closures, but moving values up out of them is tricky. Values pulled out of these state accessors must be *owned values*. Problem: **all variable reads and writes go through these closures**. Variable reads were returning `String`s, because that was just simple to implement and it worked.

  The first test I ran for this was the obvious one. I ran perf record on the arithmetic benchmark; a `((i++))` loop that hammers variable reads and writes for 100,000 iterations. The profile pointed dead at `try_get_local()`, the function that fetches a variable's value from the scope chain. 7.79% of total CPU. Inside that, almost all the time was in a single inlined `to_string()` call: every read of a variable was allocating a fresh `String`. Half a million heap allocations to count from zero to a hundred thousand.

  This behavior suggested that we needed to create our own `String`-like type that satisfied two conditions:
  1. It's an owned value.
  2. It's cheap to clone.

  Any savvy Rust devs in the audience probably had the name `Arc` appear in their head immediately on reading those two sentences. I went with `smolstr`'s `SmolStr` type, which wraps either `Arc<str>` for large strings, or a stack-allocated `[u8;24]` for small strings (under 24 bytes). I wrapped `SmolStr` in a type called `VarStr`, so that we could have full control over the interface just in case we wanted to change the inner type again later; for instance, there is potential in writing my own `SmolStr`-like struct that wraps `Rc<str>` instead of `Arc<str>` since `shed` is entirely single-threaded, but I would have to do some studying of `smolstr`'s code before tackling that.

  After landing the refactor, the arithmetic benchmark dropped from 326ms to 290ms (roughly 10-15% improvement). The `try_get_local` line vanished from the profile entirely. But the more interesting result was suite-wide: every benchmark that did variable reads (`var_lookup`, `string_ops`, `read_loop`, `func_call`, even `array`) moved down by similar percentages. One type change lifted half the suite.

  What followed though was being given a breadcrumb trail of performance issues. After one glaring problem was solved, another would immediately present itself in the `perf` data. The next issue that I ran into was that the command `Dispatcher` was kind of slow. I began investigating the structure, and started questioning the decision to pass each AST `Node` by value instead of by reference. The `Dispatcher`'s functions required an *owned `Node`*, and a consequence of this is that loops need to *clone the entire loop body on every iteration*. `Tk::clone` by itself was hogging 29% of CPU time in the arithmetic benchmark.

  Passing `Node` by value came in handy in some places, for instance being able to mutate `Node` flags/`argv` in flight, but changing the function signatures to take `&Node` seemed like a natural improvement. It would make cloning anything a deliberate and painful process, on top of getting some nice borrow checker guarantees. This worked out as I had hoped, and `Tk::clone` dropped from 29% to 9%, a 20-point swing from a single signature alteration.

  There were some places where the architecture fought this decision, since a decent amount of code assumed it was working with a mutable, owned `Node`, instead of an immutable reference. One example of this was setting the `FORK_BUILTINS` flag on pipeline nodes before executing them. For those few spots, cloning the Node and mutating the clone seemed necessary, though I'm certain there are probably improvements that can be made there to prevent it. Will require some more research.

  The breadcrumb trail continued, with the next target being `VecDeque<LabelBuilder>::clone` at 24% CPU, which is the data structure `shed` uses to hold its cool error reporting stuff. Every time the parser emitted an AST node, it was deep-cloning the parser's full error-context stack trace into the new node's metadata. Simply packing the `VecDeque` in an `Rc` was enough to wipe it off the `perf` data completely.

  In general, the `v0.29.1` update has been a massive performance win for `shed`. It's now catching up to decades-old, battle-tested shells like `bash`, `zsh`, and `fish` on benchmarks, and in some cases actually coming out on top. `shed`'s cold start for script execution is now *sub-millisecond* at around 695µs on average, about 1.76x faster than `bash`. `shed` is also winning on the pure naive-recursion fibonacci benchmark (36.2ms), surprisingly beating out even `fish` (54.8ms, 1.52x), which is the shell that [inspired](https://github.com/fish-shell/fish-shell/blob/master/src/io.rs#L167) [shed's approach](https://github.com/km-clay/shed/blob/main/src/builtin/mod.rs#L28) to optimizing command substitution.

  `shed` remains middle-of-the-pack for most of the other results, though current `perf` data shows there is still potential for improvement. The results of all of the benchmarks I ran during this optimization pass can be found below.

---

### Arithmetic

[scripts used](https://gist.github.com/km-clay/bdf70fcc78adc02a9263570f4fbfa9cd)

| shell     | time          |
|-----------|---------------|
| ***zsh*** | ***144.9ms*** |
| bash      | 212.2ms       |
| shed      | 304.3ms       |
| fish      | 1.822s        |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/arith.sh
  Time (mean ± σ):     212.2 ms ±   1.7 ms    [User: 210.9 ms, System: 0.8 ms]
  Range (min … max):   210.1 ms … 216.1 ms    14 runs

Benchmark 2: zsh   posix/arith.sh
  Time (mean ± σ):     144.9 ms ±   3.0 ms    [User: 134.9 ms, System: 9.3 ms]
  Range (min … max):   140.4 ms … 151.9 ms    21 runs

Benchmark 3: fish   fish/arith.fish
  Time (mean ± σ):      1.822 s ±  0.034 s    [User: 1.322 s, System: 0.702 s]
  Range (min … max):    1.788 s …  1.891 s    10 runs

Benchmark 4: ./shed  posix/arith.sh
  Time (mean ± σ):     304.3 ms ±   8.1 ms    [User: 302.4 ms, System: 1.0 ms]
  Range (min … max):   294.2 ms … 320.1 ms    10 runs

Summary
  zsh   posix/arith.sh ran
    1.46 ± 0.03 times faster than bash  posix/arith.sh
    2.10 ± 0.07 times faster than ./shed  posix/arith.sh
   12.58 ± 0.35 times faster than fish   fish/arith.fish
```

</details>

---

### Array Operation

[scripts used](https://gist.github.com/km-clay/9550ebf6c2c0f891be9859c952df2390)

| shell | time         |
|-------|--------------|
| ***bash***  | ***12.6ms*** |
| shed  | 13.6ms       |
| zsh   | 28.9ms       |
| fish  | 1.126s       |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/array.sh
  Time (mean ± σ):      12.6 ms ±   0.5 ms    [User: 11.3 ms, System: 1.2 ms]
  Range (min … max):    11.7 ms …  14.5 ms    237 runs

Benchmark 2: zsh   posix/array.sh
  Time (mean ± σ):      28.9 ms ±   0.6 ms    [User: 26.5 ms, System: 2.2 ms]
  Range (min … max):    27.6 ms …  31.4 ms    106 runs

Benchmark 3: fish   fish/array.fish
  Time (mean ± σ):      1.126 s ±  0.010 s    [User: 1.117 s, System: 0.006 s]
  Range (min … max):    1.110 s …  1.142 s    10 runs

Benchmark 4: ./shed  posix/array.sh
  Time (mean ± σ):      13.6 ms ±   0.4 ms    [User: 12.2 ms, System: 1.3 ms]
  Range (min … max):    12.9 ms …  16.4 ms    214 runs

Summary
  bash  posix/array.sh ran
    1.08 ± 0.05 times faster than ./shed  posix/array.sh
    2.30 ± 0.10 times faster than zsh   posix/array.sh
   89.64 ± 3.56 times faster than fish   fish/array.fish
```

</details>

---

### External Command Execution

[scripts used](https://gist.github.com/km-clay/6bae8bc5da8bac12afdb86dc6671ec48)

| shell      | time          |
|------------|---------------|
| ***bash*** | ***737.4ms*** |
| fish       | 742.4ms       |
| zsh        | 765.3ms       |
| shed       | 784.0ms       |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/external_cmds.sh
  Time (mean ± σ):     737.4 ms ±  12.7 ms    [User: 393.3 ms, System: 352.1 ms]
  Range (min … max):   718.5 ms … 754.9 ms    10 runs

Benchmark 2: zsh   posix/external_cmds.sh
  Time (mean ± σ):     765.3 ms ±   9.0 ms    [User: 389.3 ms, System: 383.9 ms]
  Range (min … max):   752.2 ms … 782.5 ms    10 runs

Benchmark 3: fish   fish/external_cmds.fish
  Time (mean ± σ):     742.4 ms ±  22.1 ms    [User: 485.7 ms, System: 256.2 ms]
  Range (min … max):   712.2 ms … 776.2 ms    10 runs

Benchmark 4: ./shed  posix/external_cmds.sh
  Time (mean ± σ):     784.0 ms ±  12.1 ms    [User: 408.6 ms, System: 384.3 ms]
  Range (min … max):   767.9 ms … 803.1 ms    10 runs

Summary
  bash  posix/external_cmds.sh ran
    1.01 ± 0.03 times faster than fish   fish/external_cmds.fish
    1.04 ± 0.02 times faster than zsh   posix/external_cmds.sh
    1.06 ± 0.02 times faster than ./shed  posix/external_cmds.sh
```

</details>

---

### Naive Recursion Fibonacci

[scripts used](https://gist.github.com/km-clay/97befd98c2112d319565f30db6501fa4)

| shell      | time         |
|------------|--------------|
| ***shed*** | ***36.2ms*** |
| fish       | 54.8ms       |
| zsh        | 461.2ms      |
| bash       | 478.0ms      |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/fib.sh
  Time (mean ± σ):     478.0 ms ±  20.4 ms    [User: 387.9 ms, System: 117.6 ms]
  Range (min … max):   439.4 ms … 492.4 ms    10 runs

Benchmark 2: zsh   posix/fib.sh
  Time (mean ± σ):     461.2 ms ±   4.6 ms    [User: 365.1 ms, System: 118.4 ms]
  Range (min … max):   457.2 ms … 471.5 ms    10 runs

Benchmark 3: fish   fish/fib.fish
  Time (mean ± σ):      54.8 ms ±   5.4 ms    [User: 37.1 ms, System: 28.0 ms]
  Range (min … max):    48.0 ms …  64.9 ms    48 runs

Benchmark 4: ./shed  posix/fib.sh
  Time (mean ± σ):      36.2 ms ±   0.8 ms    [User: 34.8 ms, System: 1.0 ms]
  Range (min … max):    34.3 ms …  39.1 ms    85 runs

Summary
  ./shed  posix/fib.sh ran
    1.52 ± 0.15 times faster than fish   fish/fib.fish
   12.76 ± 0.32 times faster than zsh   posix/fib.sh
   13.22 ± 0.64 times faster than bash  posix/fib.sh
```

</details>

---

### Function Call Overhead

[scripts used](https://gist.github.com/km-clay/e7175a4c9ea512538c63ec8839cf2986)

| shell      | time          |
|------------|---------------|
| ***bash*** | ***556.9ms*** |
| zsh        | 655.7ms       |
| shed       | 772.8ms       |
| fish       | 1.495s        |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/func_call.sh
  Time (mean ± σ):     556.9 ms ±   4.1 ms    [User: 482.7 ms, System: 72.9 ms]
  Range (min … max):   551.5 ms … 563.1 ms    10 runs

Benchmark 2: zsh   posix/func_call.sh
  Time (mean ± σ):     655.7 ms ±  11.2 ms    [User: 398.3 ms, System: 252.6 ms]
  Range (min … max):   637.7 ms … 674.2 ms    10 runs

Benchmark 3: fish   fish/func_call.fish
  Time (mean ± σ):      1.495 s ±  0.035 s    [User: 1.123 s, System: 0.482 s]
  Range (min … max):    1.456 s …  1.572 s    10 runs

Benchmark 4: ./shed  posix/func_call.sh
  Time (mean ± σ):     772.8 ms ±  14.9 ms    [User: 702.3 ms, System: 67.7 ms]
  Range (min … max):   749.8 ms … 795.6 ms    10 runs

Summary
  bash  posix/func_call.sh ran
    1.18 ± 0.02 times faster than zsh   posix/func_call.sh
    1.39 ± 0.03 times faster than ./shed  posix/func_call.sh
    2.69 ± 0.07 times faster than fish   fish/func_call.fish
```

</details>

---

### Filesystem Globbing

[scripts used](https://gist.github.com/km-clay/e29669bdaf64ce50d87b53c64ed2af1d)

| shell     | time          |
|-----------|---------------|
| ***zsh*** | ***128.6ms*** |
| shed      | 154.5ms       |
| bash      | 167.2ms       |
| fish      | 929.9ms       |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/glob.sh
  Time (mean ± σ):     167.2 ms ±   3.1 ms    [User: 155.3 ms, System: 11.2 ms]
  Range (min … max):   162.8 ms … 174.1 ms    17 runs

Benchmark 2: zsh   posix/glob.sh
  Time (mean ± σ):     128.6 ms ±   2.0 ms    [User: 115.0 ms, System: 12.9 ms]
  Range (min … max):   124.5 ms … 131.6 ms    23 runs

Benchmark 3: fish   fish/glob.fish
  Time (mean ± σ):     929.9 ms ±  20.4 ms    [User: 632.2 ms, System: 403.8 ms]
  Range (min … max):   902.5 ms … 958.0 ms    10 runs

Benchmark 4: ./shed  posix/glob.sh
  Time (mean ± σ):     154.5 ms ±   3.1 ms    [User: 142.9 ms, System: 10.7 ms]
  Range (min … max):   149.0 ms … 160.7 ms    19 runs

Summary
  zsh   posix/glob.sh ran
    1.20 ± 0.03 times faster than ./shed  posix/glob.sh
    1.30 ± 0.03 times faster than bash  posix/glob.sh
    7.23 ± 0.19 times faster than fish   fish/glob.fish
```

</details>

---

### Here-Document Redirection

[scripts used](https://gist.github.com/km-clay/f8630eb55cdceb59ca12fbdbc306a453)
<sub>note: fish doesn't have heredocs, so its not in this benchmark</sub>

| shell      | time          |
|------------|---------------|
| ***bash*** | ***500.0ms*** |
| shed       | 527.8ms       |
| zsh        | 551.2ms       |
| fish       | N/A           |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/heredoc.sh
  Time (mean ± σ):     500.0 ms ±  15.9 ms    [User: 312.0 ms, System: 198.4 ms]
  Range (min … max):   482.4 ms … 520.5 ms    10 runs

Benchmark 2: zsh   posix/heredoc.sh
  Time (mean ± σ):     551.2 ms ±   7.8 ms    [User: 326.4 ms, System: 233.1 ms]
  Range (min … max):   537.6 ms … 560.8 ms    10 runs

Benchmark 3: ./shed  posix/heredoc.sh
  Time (mean ± σ):     527.8 ms ±   8.7 ms    [User: 329.6 ms, System: 207.2 ms]
  Range (min … max):   520.1 ms … 546.5 ms    10 runs

Summary
  bash  posix/heredoc.sh ran
    1.06 ± 0.04 times faster than ./shed  posix/heredoc.sh
    1.10 ± 0.04 times faster than zsh   posix/heredoc.sh
```

</details>

---

### Script Parse Time

this benchmark tests the parse time of the `git-completions.sh` script. I don't know if there's a fish equivalent, so it's not tested here.

| shell      | time        |
|------------|-------------|
| ***bash*** | ***2.8ms*** |
| zsh        | 3.6ms       |
| shed       | 7.3ms       |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  -n posix/parse_only.sh
  Time (mean ± σ):       2.8 ms ±   0.3 ms    [User: 2.3 ms, System: 0.4 ms]
  Range (min … max):     2.5 ms …   4.1 ms    754 runs

Benchmark 2: zsh   -n posix/parse_only.sh
  Time (mean ± σ):       3.6 ms ±   0.2 ms    [User: 2.7 ms, System: 0.9 ms]
  Range (min … max):     3.2 ms …   4.6 ms    818 runs

Benchmark 3: ./shed  -n posix/parse_only.sh
  Time (mean ± σ):       7.3 ms ±   0.6 ms    [User: 4.6 ms, System: 2.5 ms]
  Range (min … max):     6.0 ms …   9.6 ms    412 runs

Summary
  bash  -n posix/parse_only.sh ran
    1.28 ± 0.13 times faster than zsh   -n posix/parse_only.sh
    2.58 ± 0.32 times faster than ./shed  -n posix/parse_only.sh
```

</details>

---

### Tight `read` Loop

[scripts used](https://gist.github.com/km-clay/49fe890c3090af938c7fcb29579d6f0b)

| shell      | time         |
|------------|--------------|
| ***bash*** | ***26.3ms*** |
| zsh        | 29.8ms       |
| shed       | 36.1ms       |
| fish       | 127.6ms      |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/read_loop.sh
  Time (mean ± σ):      26.3 ms ±   0.6 ms    [User: 22.9 ms, System: 3.1 ms]
  Range (min … max):    25.0 ms …  28.0 ms    117 runs

Benchmark 2: zsh   posix/read_loop.sh
  Time (mean ± σ):      29.8 ms ±   0.6 ms    [User: 18.9 ms, System: 10.6 ms]
  Range (min … max):    28.3 ms …  31.6 ms    102 runs

Benchmark 3: fish   fish/read_loop.fish
  Time (mean ± σ):     127.6 ms ±  10.8 ms    [User: 82.5 ms, System: 57.2 ms]
  Range (min … max):   113.5 ms … 146.8 ms    25 runs

Benchmark 4: ./shed  posix/read_loop.sh
  Time (mean ± σ):      36.1 ms ±   1.0 ms    [User: 27.8 ms, System: 8.0 ms]
  Range (min … max):    34.4 ms …  40.5 ms    84 runs

Summary
  bash  posix/read_loop.sh ran
    1.13 ± 0.04 times faster than zsh   posix/read_loop.sh
    1.37 ± 0.05 times faster than ./shed  posix/read_loop.sh
    4.86 ± 0.43 times faster than fish   fish/read_loop.fish
```

</details>

---

### Variable Setting Benchmark

[scripts used](https://gist.github.com/km-clay/fa8a454ee34073c6fbe647a9b926597c)

| shell      | time          |
|------------|---------------|
| ***fish*** | ***158.9ms*** |
| bash       | 304.7ms       |
| zsh        | 399.1ms       |
| shed       | 425.9ms       |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/set_long.sh
  Time (mean ± σ):     304.7 ms ±   3.5 ms    [User: 303.1 ms, System: 0.8 ms]
  Range (min … max):   298.4 ms … 309.4 ms    10 runs

Benchmark 2: zsh   posix/set_long.sh
  Time (mean ± σ):     399.1 ms ±  10.0 ms    [User: 223.2 ms, System: 171.1 ms]
  Range (min … max):   380.5 ms … 416.8 ms    10 runs

Benchmark 3: fish   fish/set_long.fish
  Time (mean ± σ):     158.9 ms ±   1.7 ms    [User: 148.9 ms, System: 9.9 ms]
  Range (min … max):   156.7 ms … 162.5 ms    18 runs

Benchmark 4: ./shed  posix/set_long.sh
  Time (mean ± σ):     425.9 ms ±   5.2 ms    [User: 424.0 ms, System: 0.8 ms]
  Range (min … max):   417.3 ms … 435.3 ms    10 runs

Summary
  fish   fish/set_long.fish ran
    1.92 ± 0.03 times faster than bash  posix/set_long.sh
    2.51 ± 0.07 times faster than zsh   posix/set_long.sh
    2.68 ± 0.04 times faster than ./shed  posix/set_long.sh
```

</details>

---

### String Operations

[scripts used](https://gist.github.com/km-clay/5b56400e8a517907e8b677a8560e0a16)

| shell     | time          |
|-----------|---------------|
| ***zsh*** | ***435.3ms*** |
| bash      | 537.4ms       |
| shed      | 666.5ms       |
| fish      | 3.780s        |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/string_ops.sh
  Time (mean ± σ):     537.4 ms ±   4.7 ms    [User: 534.8 ms, System: 1.3 ms]
  Range (min … max):   531.7 ms … 546.9 ms    10 runs

Benchmark 2: zsh   posix/string_ops.sh
  Time (mean ± σ):     435.3 ms ±   6.1 ms    [User: 424.9 ms, System: 9.3 ms]
  Range (min … max):   429.6 ms … 446.9 ms    10 runs

Benchmark 3: fish   fish/string_ops.fish
  Time (mean ± σ):      3.780 s ±  0.054 s    [User: 2.793 s, System: 1.383 s]
  Range (min … max):    3.689 s …  3.871 s    10 runs

Benchmark 4: ./shed  posix/string_ops.sh
  Time (mean ± σ):     666.5 ms ±  11.3 ms    [User: 663.3 ms, System: 1.1 ms]
  Range (min … max):   651.4 ms … 685.8 ms    10 runs

Summary
  zsh   posix/string_ops.sh ran
    1.23 ± 0.02 times faster than bash  posix/string_ops.sh
    1.53 ± 0.03 times faster than ./shed  posix/string_ops.sh
    8.68 ± 0.17 times faster than fish   fish/string_ops.fish
```

</details>

---

### Cold Start

[scripts used](https://gist.github.com/km-clay/854f44afc9b5f411a921e698c81e01c7)

| shell      | time          |
|------------|---------------|
| ***shed*** | ***695.8µs*** |
| bash       | 1.2ms         |
| zsh        | 1.7ms         |
| fish       | 6.3ms         |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/true.sh
  Time (mean ± σ):       1.2 ms ±   0.1 ms    [User: 0.9 ms, System: 0.3 ms]
  Range (min … max):     1.1 ms …   2.1 ms    1658 runs

Benchmark 2: zsh   posix/true.sh
  Time (mean ± σ):       1.7 ms ±   0.2 ms    [User: 1.1 ms, System: 0.5 ms]
  Range (min … max):     1.4 ms …   5.5 ms    1722 runs

Benchmark 3: fish   fish/true.fish
  Time (mean ± σ):       6.3 ms ±   0.3 ms    [User: 3.8 ms, System: 2.6 ms]
  Range (min … max):     5.6 ms …   7.4 ms    476 runs

Benchmark 4: ./shed  posix/true.sh
  Time (mean ± σ):     695.8 µs ± 106.5 µs    [User: 378.7 µs, System: 247.7 µs]
  Range (min … max):   575.4 µs … 1361.8 µs    3837 runs

Summary
  ./shed  posix/true.sh ran
    1.76 ± 0.32 times faster than bash  posix/true.sh
    2.39 ± 0.43 times faster than zsh   posix/true.sh
    9.08 ± 1.45 times faster than fish   fish/true.fish
```

</details>

---

### Variable Mixed Read/Write

[scripts used](https://gist.github.com/km-clay/f92dfb485f53f22d2acf70201765086c)

| shell     | time          |
|-----------|---------------|
| ***zsh*** | ***158.2ms*** |
| bash      | 251.8ms       |
| shed      | 352.9ms       |
| fish      | 2.562s        |

<details markdown="1">
<summary>full hyperfine output</summary>

```bash
Benchmark 1: bash  posix/var_lookup.sh
  Time (mean ± σ):     251.8 ms ±   1.9 ms    [User: 250.5 ms, System: 0.6 ms]
  Range (min … max):   248.2 ms … 253.9 ms    11 runs

Benchmark 2: zsh   posix/var_lookup.sh
  Time (mean ± σ):     158.2 ms ±   2.3 ms    [User: 148.0 ms, System: 9.6 ms]
  Range (min … max):   153.7 ms … 162.5 ms    19 runs

Benchmark 3: fish   fish/var_lookup.fish
  Time (mean ± σ):      2.562 s ±  0.041 s    [User: 1.802 s, System: 1.060 s]
  Range (min … max):    2.503 s …  2.628 s    10 runs

Benchmark 4: ./shed  posix/var_lookup.sh
  Time (mean ± σ):     352.9 ms ±   3.5 ms    [User: 350.2 ms, System: 1.1 ms]
  Range (min … max):   348.7 ms … 358.6 ms    10 runs

Summary
  zsh   posix/var_lookup.sh ran
    1.59 ± 0.03 times faster than bash  posix/var_lookup.sh
    2.23 ± 0.04 times faster than ./shed  posix/var_lookup.sh
    16.19 ± 0.35 times faster than fish   fish/var_lookup.fish
```

</details>
