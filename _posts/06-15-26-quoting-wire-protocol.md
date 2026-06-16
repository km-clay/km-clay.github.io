---
layout: post
title: Shell Quoting as a Wire Protocol
date: 2026-06-15
---

  Yesterday, I wrote a post called ["Quoted Data Streams for Shell Scripts"](https://km-clay.github.io/2026/06/14/shell-structured-data.html), which detailed the `quote` and `unquote` builtins I implemented for [shed](https://github.com/km-clay/shed). I thought I was going pretty in-depth with what these builtins were capable of doing, but it turns out this rabbit hole is actually much deeper than I had initially anticipated.

  Today I was planning on getting started on `shed`'s included `git` completion script. Every shell needs one of these, and `shed` is no exception. I was looking around through existing implementations, and found this snippet in `fish`'s git completions:
```bash
function __fish_git_local_branches
  # This is much faster than using `git branch` and avoids dealing with localized "detached HEAD" messages.
  # We intentionally only sort local branches by recency. See discussion in #9248.
  __fish_git for-each-ref --format='%(refname:strip=2)%09Local Branch' --sort=-committerdate refs/heads/ 2>/dev/null
end
```

  `%09` in the --format arg is the tab character. So the format produces something like
```
main  Local Branch
develop   Local Branch
feature/foo Local Branch
```

  Fish's completion system then splits each line on tab, left side is the candidate, right is the description. This works because tabs cannot appear in branch names, so it's safe to use it as a delimiter here. I realized though, that if I was going to translate this to `shed`, I might be able to use `quote` to do the heavy lifting for field delimiting in general. Here's the translation I came up with:

```bash
__git_local_branches() {
  git for-each-ref --format='%(refname:strip=2)' --sort=-committerdate refs/heads/ \
  | while IFS= read -r name; do
    quote "$name" "Local Branch"
  done
}
```

  Which produces this kind of output:
```bash
$ __git_local_branches
main 'Local Branch'
visual_block 'Local Branch'
immutable-ast 'Local Branch'
autoload 'Local Branch'
```

  This will be very easy to work with using `read -q`, like this:
```bash
  $ __git_local_branches | while IFS= read -q branch desc; do
2 |   echo "$branch is a $desc"
3 | done
main is a Local Branch
visual_block is a Local Branch
immutable-ast is a Local Branch
autoload is a Local Branch
```

  `read -q` decodes quoted fields the same way that `unquote` does, instead of using newlines or delimiter bytes passed with `-d`. I found this to be somewhat verbose though. The first line doesn't read super cleanly, and I figured if this "quoting protocol" was going to be used often, there should probably be a more elegant way to consume it. I started thinking about better ways to actually work with the quoted data. Take a look at this function:
```bash
qmap() {
  local body="$1"
  eval "__qmap_lambda() { $body; }"
  defer unset -f __qmap_lambda

  while read -q -a fields; do
    __qmap_lambda "${fields[@]}"
  done
}
```

  `eval` is used to create a temporary function that uses `$1` as the body, and `defer` is used to schedule a cleanup of that function after the scope is dropped. Each record is then passed to the given "lambda".

  I was skeptical that something like this could actually work well, but lo and behold:
```bash
$ __git_local_branches | qmap 'echo $1 is a $2'
main is a Local Branch
visual_block is a Local Branch
immutable-ast is a Local Branch
autoload is a Local Branch
```

  Something worth noting is that `qmap` is a *shell function*, not a builtin. The `quote`/`unquote`/`read -q` substrate turned out to be entirely sufficient for expressing this kind of structured-data tooling without any language extensions. The shell functions demonstrated in this blog post are probably just the tip of the iceberg for what's possible with this, and it's all entirely user-extensible.

  Anyway, instead of needing a whole word-salad while loop to map logic to the fields, we can actually express the same thing with a single command and a single argument: `qmap 'echo $1 is a $2'`. We now have primitives for outputting structured data (`quote`, `split`, etc), and we now have a *workable pattern* for creating primitives that consume and operate on it. Immediately upon seeing that this pattern could work, I thought of `nushell`'s approach to manipulating structured data. What if we could actually implement something like `where`? As it turns out, it's not too difficult:
```bash
qwhere() {
  local body="$1"
  eval "__qwhere_lambda() { $body; }"
  defer unset -f __qwhere_lambda

  while IFS= read -r line; do
    local -a fields
    unquote -a fields "$line"
    if __qwhere_lambda "${fields[@]}"; then
      printf '%s\n' "$line"
    fi
  done
}
```

  Same pattern, only this time around we save the raw line and call `unquote` on it directly, packing it into the `$fields` array. We then give these fields to `__qwhere_lambda`, and if that function returns 0, `$line` is passed through untouched. The result is a cleanly filtered array of records:

```bash
  # before
  $ for i in {1..5}; do
2 |      quote "${records[@]}"
3 |      rotate records # ring buffer rotation
4 | done
'foo bar' 'biz buzz' 'bam bop'
'biz buzz' 'bam bop' 'foo bar'
'bam bop' 'foo bar' 'biz buzz'
'foo bar' 'biz buzz' 'bam bop'
'biz buzz' 'bam bop' 'foo bar'

  # after
  $ for i in {1..5}; do
2 |      quote "${records[@]}"
3 |      rotate records
4 | done | qwhere '[[ "$1" == b* ]]' # first field starts with 'b'
'biz buzz' 'bam bop' 'foo bar'
'bam bop' 'foo bar' 'biz buzz'
'biz buzz' 'bam bop' 'foo bar'
```

  So, this is nice and all, but now the burning question: does this compose with standard utilities? The next function I implemented was `qpipe`:
```bash
qpipe() {
  while read -q -a fields; do
    local -a transformed=()

    while IFS= read -r line; do
      push transformed "$line"
    done < <(printf '%s\n' "${fields[@]}" | "$@")

    [ "${#transformed[@]}" -gt 0 ] && quote "${transformed[@]}"
  done
}
```

  This function reads all of the records given to it, and pipes each one to a given command. The output is then emitted using `quote` to maintain structure. Turns out that it works as expected:
```bash
  $ for i in {1..5}; do
2 |      quote "${arr[@]}"
3 |      rotate arr
4 | done | qwhere '[[ "$1" == b* ]]' | qpipe tr a-z A-Z # lowercase to uppercase
'BAM BOP' 'FOO BAR' 'BIZ BUZZ'
'BIZ BUZZ' 'BAM BOP' 'FOO BAR'
'BAM BOP' 'FOO BAR' 'BIZ BUZZ'
```

  This whole time I've been showing custom `shed` functions composing with each other, which is all well and good, but it's not really a "protocol" if only one program can work with the data. Let's see what happens if we throw in some coreutils:
```bash
  $ for i in {1..6}; do
2 |      quote "${arr[@]}"
3 |      rotate arr
4 |  done | sort | tr a-z A-Z | sed 's/BUZZ/REPLACED/'
'BAM BOP' 'FOO BAR' 'BIZ REPLACED'
'BAM BOP' 'FOO BAR' 'BIZ REPLACED'
'BIZ REPLACED' 'BAM BOP' 'FOO BAR'
'BIZ REPLACED' 'BAM BOP' 'FOO BAR'
'FOO BAR' 'BIZ REPLACED' 'BAM BOP'
'FOO BAR' 'BIZ REPLACED' 'BAM BOP'
```

  This pipeline involves three external programs: `sort`, `tr`, and `sed`. None of them know anything about `shed`, quoted records, or any of the substrate. They strictly process lines of bytes, which is exactly what the protocol *guarantees* they'll see. The unix toolchain participates in these quoted pipelines for free.

```bash
  $ for i in {1..6}; do
2 |      quote "${arr[@]}"
3 |      rotate arr
4 |  done | \
5 |  sort | \
6 |  uniq | \
7 |  tr a-z A-Z | \
8 |  sed 's/BUZZ/REPLACED/' | \
9 |  qmap 'quote "$@" | unquote -0 | xargs -0 -I {} echo field: {}; echo DONE'
field: BAM BOP
field: FOO BAR
field: BIZ REPLACED
DONE
field: BIZ REPLACED
field: BAM BOP
field: FOO BAR
DONE
field: FOO BAR
field: BIZ REPLACED
field: BAM BOP
DONE
```

  The protocol holds cleanly even under relatively intense operation like this. A 6-stage pipeline, where the last stage is itself a 3-stage pipeline executed per record, and 5 of the tools involved are standard coreutils.

  Also worth noting: here, we use `quote "$@"` at the start of the qmap function because they get turned back into normal splittable function arguments at the boundary. We re-encode them, then decode using `unquote -0` and pass off to `xargs -0`, which also speaks the `-0` language.

  For an example of a real application, let's try CSV parsing. CSV is deceptively annoying to parse; quoted fields with commas, embedded newlines, escaped quotes, to name a few things. However, it's also where this shell quoting protocol really pulls weight, since every one of those pain points is exactly what `quote` exists to handle on the wire.
  Here's the parser I wrote ([source](https://gist.github.com/km-clay/f3dadae3cfd1fe1bc26c5ac41f6f8521)), and here's the disgusting file it parses ([source](https://gist.github.com/km-clay/e7c8ddfc513d015390e42b9630ab0845))

```bash
$ parse_csv < ./ref/csv_from_hell.csv
id name email priority status description tags created_at
1 'Smith, John' john@example.com high open $'Cannot log in to account.\n\nTried clearing cache, still broken.' 'auth;login' 2026-06-10T14:23:00Z
2 'O'\''Brien, Mary' mary.obrien@example.com medium resolved 'User reported: "Page loads slowly"' 'performance;ui' 2026-06-11T09:15:32Z
3 'Müller, Hans' hans@example.de low open 'Quick question about billing — does the discount apply?' billing 2026-06-11T16:42:08Z
4 'Tanaka, 田中' tanaka@example.jp critical open $'Server returned 500 error.\nStack trace:\n  at handler.js:42\n  at server.js:108\nNeed urgent fix.' 'backend;500;urgent' 2026-06-12T03:01:55Z
5 '' empty.name@example.com low closed 'Field test: comma, semicolon; pipe|; backslash\; quote"end' test 2026-06-12T11:30:00Z
6 'Wong, Wei-Chen' wong@example.cn high open $'Customer says: "I love the new feature, but the button placement is confusing."\n\nSpecifically the "Save" button.' 'ux;feedback' 2026-06-13T08:00:00Z
7 'García, José' jose@example.es medium in_progress 'Resolving login issue for account #12345. ETA: 2 hours.' 'auth;in_progress' 2026-06-13T15:22:11Z
8 Anonymous user-12345@example.com low open $'Multi-paragraph report:\n\nPara 1: Cannot edit profile.\nPara 2: Save button greyed out.\nPara 3: Logout doesn\'t work either.' 'profile;multiple' 2026-06-13T18:00:00Z
9 'Edge Case' edge@example.com low open '"' test 2026-06-14T00:00:00Z
```

  Correctly splits it into quoted fields. Let's try running this through the adapter functions... I think filtering to only see "high" or "critical" priority records would be useful.

```bash
  $ parse_csv < ./ref/csv_from_hell.csv \
2 |  | qwhere '[ "$4" = "high" ] || [ "$4" = "critical" ]'
1 'Smith, John' john@example.com high open $'Cannot log in to account.\n\nTried clearing cache, still broken.' 'auth;login' 2026-06-10T14:23:00Z
4 'Tanaka, 田中' tanaka@example.jp critical open $'Server returned 500 error.\nStack trace:\n  at handler.js:42\n  at server.js:108\nNeed urgent fix.' 'backend;500;urgent' 2026-06-12T03:01:55Z
6 'Wong, Wei-Chen' wong@example.cn high open $'Customer says: "I love the new feature, but the button placement is confusing."\n\nSpecifically the "Save" button.' 'ux;feedback' 2026-06-13T08:00:00Z
```

  `qwhere` handles this elegantly. Now then, let's limit the output to specific fields, since this output is pretty noisy.

```bash
  $ parse_csv < ./ref/csv_from_hell.csv \
2 |  | qwhere '[ "$4" = "high" ] || [ "$4" = "critical" ]' \
3 |  | qselect 1 2 6
1 'Smith, John' $'Cannot log in to account.\n\nTried clearing cache, still broken.'
4 'Tanaka, 田中' $'Server returned 500 error.\nStack trace:\n  at handler.js:42\n  at server.js:108\nNeed urgent fix.'
6 'Wong, Wei-Chen' $'Customer says: "I love the new feature, but the button placement is confusing."\n\nSpecifically the "Save" button.'
```

  `qselect` is another one of these "adapter" functions. It lets you select specific fields to include in the output. In this case, `id`, `name`, and `description`. For the finishing touch, let's take this output and use `qmap` to format it.

```bash
  $ parse_csv < ./ref/csv_from_hell.csv \
2 |  | qwhere '[ "$4" = "high" ] || [ "$4" = "critical" ]' \
3 |  | qselect 1 2 6 \
4 |  | qmap 'echo "[$1] $2 - $3"; echo "---"'
[1] Smith, John - Cannot log in to account.

Tried clearing cache, still broken.
---
[4] Tanaka, 田中 - Server returned 500 error.
Stack trace:
  at handler.js:42
  at server.js:108
Need urgent fix.
---
[6] Wong, Wei-Chen - Customer says: "I love the new feature, but the button placement is confusing."

Specifically the "Save" button.
---
```

  Very clean. The structure of the data has been maintained completely through 3 different transformer functions.

  This "shell quote protocol" has actually enabled something that is very difficult to achieve otherwise when working with data in shell scripts: we can enforce a shape for the data we are passing around, and actually trust the input we are receiving. The contract is as follows:

  1. Records are separated by newlines. (newlines in content are wrapped and escaped by ANSI-C quotes `$'...'`)
  2. Fields are separated by spaces.
  3. If you see a space or newline, it is *guaranteed* to be a separator.

  It will require more testing to see if this actually ends up being as useful as I think it is. From these developments though, it is looking very promising. It feels as though an entire class of "ok time to use Python I guess" moments has completely evaporated.
