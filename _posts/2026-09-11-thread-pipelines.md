---
layout: post
title: Multi-threaded Pipelines
date: 2026-09-11
---

In my last blog post I detailed the new virtual file descriptor table that shed uses internally for I/O management. The idea is to manage our own FD table, so that we can basically do internal I/O dispatch the same way that the kernel does, and avoid forking when we don't really need to. I also mentioned that it opens up the possibility of running *multi-threaded pipelines*.

It is an idea that sounds like a free win on paper, something that anyone would snatch up without a second thought, but it turned out to be far more complicated than I initially anticipated. I'll get to that later though, for now let's enjoy our last moments of naive optimism.

One of the main payoffs of this model is that we could now spawn threads to *simulate* forking for builtin commands, and we could do this *unconditionally* no matter where the builtin was in the pipeline. Previously, shed's internal pipelines were limited in these regards:
* They could only run sequentially; all command output had to be buffered before moving to the next stage
* They could only internalize a *tail sequence* of builtins, for instance, in this pipeline:
```sh
builtin | command | builtin | builtin | builtin
```
The first two commands would fork, since shed wouldn't be able to both execute the first builtin *and* wait on the second command at the same time. So, in essence, two pipelines would actually run here: an external pipeline (`builtin | command`) that feeds an internal pipeline (`builtin | builtin | builtin`)

## ThreadSink

With the introduction of the `Sink` trait and the `ThreadSink` variant however, builtins could now be dispatched as background threads that run concurrently with external commands, and stream/consume input the exact same way. Each pipeline segment could theoretically use the smallest amount of work required to perform the task at hand: consuming input, executing some code, streaming output.

`ThreadSink` was a particularly interesting struct to implement. The main structural decision was between using `std::sync::mpsc::channel()` to create the cross-thread FIFO, or handrolling our own pipes. Ultimately, I went with handrolling our own, for two reasons:
1. It seemed like a good opportunity to learn how stuff like this works under the hood
2. I am somewhat superstitious when it comes to using specialized `std` types in this codebase for one reason or another. More often than not, wrestling with the abstraction leads to more trouble than it is worth.

And so, we ended up with `ThreadSink`, a cross-thread FIFO that can correctly block writes with backpressure and uses a `Condvar` to notify readers when new data is posted. Pretty simple stuff but simple is usually best for things like this, in my experience. The backing type for this is a simple `VecDeque<u8>` with a write limit and flags for whether or not the reader/writer are still open. The implementation details are [here](https://github.com/km-clay/shed/blob/51cc665119273f24315376732e5e9475b6150e75/src/procio.rs#L857) if you'd like to take a look.

## Profiling

After the implementation for this finally landed, and we actually had concurrent internal pipelines, the obvious next step was to go see how much of a performance win I just got. Because surely everything had to have been sped up by this change, right? I ran the perf suite and the results are not what you want to see after 8 straight hours of gutting and replacing your shell's internals.

#### shed mean runtime, milliseconds:


| benchmark | before | after | delta |
|---|--:|--:|--:|
| subshell | 3633 | 4884 | +34% |
| fib | 448 | 563 | +26% |
| loop | 1160 | 1443 | +24% |
| var_assign | 721 | 879 | +22% |
| string_concat | 76 | 92 | +21% |
| param_expand | 799 | 967 | +21% |
| func_call | 1243 | 1485 | +20% |
| arith | 773 | 917 | +19% |
| cmd_sub | 465 | 530 | +14% |


A performance regression of ~20-25% on average across the board. What could be causing this? Well, turns out that replacing `Rc` with `Arc` is actually that big of a deal for very hot loops. After installing the logic required to choose between dispatching threads or forks, every command suddenly needed to become thread capable, and even single commands were moving through this machinery since shed sees a standalone command as "a pipeline of length 1".

This ended up as more of a pyrrhic victory than a failure though. On the `builtin_pipe` benchmark, shed was finishing in ~1.33s vs bash/zsh at ~3.2-3.5s, so the concurrency *was* paying off at the very least. On top of this, builtins now had an actual entry on the job table as a thread `JoinHandle`, which was the counterpart to a forked process' `Pid`. Background threads could now be waited on and reaped in the same way as child processes, just by calling `join()` instead of `waitpid()`. Even if performance regressed somewhat, this was ultimately a win from an architectural standpoint. Builtins are now treated the exact same way that externals are, all while *never* needing to fork a new process.

Over the course of the next couple of weeks, I *was* able to slowly claw back most of the performance cost of this refactor, and shed in its current state is actually *faster* in many regards than it was before the refactor. This required a nearly wholesale refactor of the shell's expansion machinery and argument handling, which I plan to cover in a future blog post.


| benchmark | before | now | delta |
|---|--:|--:|--:|
| cmd_sub | 465 | 268 | -42% |
| fib | 448 | 366 | -18% |
| param_expand | 799 | 741 | -7% |
| var_assign | 721 | 684 | -5% |
| string_concat | 76 | 72.5 | -5% |
| arith | 773 | 771 | ~even |
| loop | 1160 | 1193 | +3% |
| func_call | 1243 | 1312 | +6% |
| subshell | 3633 | 4545 | +25% |


On "fork"-heavy workloads like the `cmd_sub` benchmark, we now see pretty dramatic performance gains. The last place to really dig into is the `subshell` benchmark, which does a lot of process and scope setup/teardown that could likely be made more efficient somehow.

## Conclusion

Overall, I think this redesign has landed quite strongly, and I'd like to dedicate some space on this post to the design ideas from other shells that inspired my architectural decisions for both the FD table and the pipeline threading model:

* **ksh93 + sfio** - `ksh93` (the korn shell), is built on sfio ("Safe/Fast I/O"), a userspace stdio replacement whose streams can be memory-backed, similar to our internal `Sink` variants. `ksh93`'s builtins are written against sfio streams rather than raw fds, effectively implementing an I/O abstraction layer similar to what I designed here.
* **busybox (NOFORK/NOEXEC applets)** - runs coreutils-equivalent builtins in-process to avoid fork/exec. Basically the main prior-art for shed's design philosophy, which is to expose primitives as shell builtins rather than relying on external binaries.
* **PowerShell / nushell** - fully internal, no kernel pipes between stages, streaming pipeline. This is basically the model taken to its theoretical limit. Abandons raw byte streams in favor of structured data pipelines. Shed attempts to allow for both by providing the `quote`/`unquote` builtins for maintaining byte structure across internal pipelines.
* **fish** ([`IoChain`](https://github.com/fish-shell/fish-shell/blob/35fd72ad034e8e75451debe29535240bbd689980/src/io.rs#L510)/[`IoBufferfill`](https://github.com/fish-shell/fish-shell/blob/35fd72ad034e8e75451debe29535240bbd689980/src/io.rs#L302)) - fish builtins run in the parent, but capture still allocates a real kernel pipe and a drain thread. The `IoMode` enum is my main and original inspiration for even trying something like this in the first place. The polymorphic I/O interface is certainly the right idea.
