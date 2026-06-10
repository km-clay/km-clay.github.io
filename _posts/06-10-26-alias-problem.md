---
layout: post
title: The Alias Problem
date: 2026-06-09
---

  `shed` uses a strategy for lexing/parsing that is unique among the shells that I have studied, and is technically a POSIX deviation. While POSIX specifies that input should be tokenized/parsed/executed one command at a time, `shed` actually consumes the entire input, constructs an AST, then executes the whole thing at once.

  This decision comes with benefits and trade-offs. The biggest benefit is the fact that syntax errors are caught *before* any execution happens. This makes scripts written in `shed` atomic. You won't have operations that are left in a half-completed state because the execution stops halfway through due to a syntax error. The entire script is either executable or it's not.

  Another benefit is that syntax error reporting can be almost compiler-grade in terms of granularity. The parser actually has what's known as a "panic mode", when it hits an error, it does not stop the parse there. It proceeds to the next separator (semicolon or newline), dropping every token along the way, and then continues parsing. This allows `shed`'s parser to catch not just the first syntax error, but *all* of the syntax errors present in the script. This is good news for any potential future LSP tooling for `shed` scripting, as with existing shell syntax checkers like `bashls` or `shellcheck`, parsing stops at the first syntax error found. Take this basic script for example:

```bash
#!/usr/bin/env bash

echo this
echo is
echo executed

if foo; then fi # syntax error, line 7

echo this
echo is
echo not

if foo; then fi # also a syntax error, line 13
```

  In this script, my editor's diagnostics show me this:
```
   ~/misc/tests/scripts/  4
  └╴  syntax.sh  4
    ├╴E Couldn't parse this then clause. Fix to allow more checks. shellcheck (SC1073) [7, 9]
    ├╴E Can't have empty then clauses (use 'true' as a no-op). shellcheck (SC1048) [7, 14]
    ├╴E Unexpected keyword/token. Fix any mentioned problems and try again. shellcheck (SC1072) [7, 16]
    └╴I The mentioned syntax error was in this if expression. shellcheck (SC1009) [7, 1]
```

  Only one syntax error is reported. But there are two faulty `if` statements! In contrast, `shed`'s error reporting on the same script:
```bash
Error: Parse Error
   ╭─[ ~/misc/tests/scripts/syntax.sh:7:1 ]
   │
 7 │ if foo; then fi
   │ ───────┬───────
   │        ╰───────── Expected a command after 'then'
───╯

Error: Parse Error
    ╭─[ ~/misc/tests/scripts/syntax.sh:13:1 ]
    │
 13 │ if foo; then fi
    │ ───────┬───────
    │        ╰───────── Expected a command after 'then'
────╯
```

  You can see how this would be more ergonomic for writing longer scripts.

  This design decision, however, does not come without its own trade-offs. For instance, the problem that this article is named after, the problem of **shell aliases**.

  Take a look at this script:
```bash
#!/usr/bin/env zsh

alias foo="echo bar"

foo
```

  In `zsh`, this will print "bar", because aliases can be defined and expanded in the same script that they are defined in. Same with `bash`, if you set `shopt -s expand_aliases`. These shells are capable of this because they parse commands one at a time. However, even these shells have limits. For instance:
```bash
#!/usr/bin/env zsh

{
  alias foo="echo bar"

  foo
}
```

  In `zsh` and `bash`, this script now suddenly throws an error saying that `foo` is not found. This is because the entire brace group is lexed, parsed, expanded, and executed as a single logical unit.

  For `shed` though, the limitations run a bit deeper. For the original script:
```bash
Error: Not Found
   ╭─[ ~/misc/tests/scripts/alias.sh:5:1 ]
   │
 5 │ foo
   │ ─┬─
   │  ╰─── command not found
───╯
```

  No dice. This is because, due to `shed`'s atomic lexing/parsing model, shell aliases *must be expanded before lexing even begins*. This problem is a fundamental drawback of using an atomic parsing model, but I personally believe the trade-off is worth it. Better diagnostics and safer scripts are, in my opinion, higher value than being able to expand aliases in the script that defines them. There are workarounds for it too, like just defining a shell function instead of an alias.
