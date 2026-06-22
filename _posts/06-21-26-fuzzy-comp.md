---
layout: post
title: Generalizing Shed Features
date: 2026-06-21
---

I think `shed`'s current featureset can be considered pretty close to complete, at least from a *general* usability standpoint. There are many features that are basically required for a shell in 2026, like tab completion, and command history. Some others, like syntax highlighting and autosuggestions, are also starting to pick up steam among this type of program. While I was implementing these general features, there were a few experimental features I wrote as well. These features kind of stuck around because I thought they were cool at the time, and they just work. The features I'm mainly talking about are:
* The `screensaver_cmd` shopt
* The builtin `fzf` tab completion thing.

As `shed` has continued growing as a program, these two features have started feeling like they have less and less of a real place in the codebase. Both of them make sense for a small program with few extensibility endpoints, I mean one is just a hardcoded UI element and the other is "execute this string after N seconds", but in the context of present-day `shed`, it is hard to look at these features and not wonder if these could just be merged into some other existing subsystem.

The `screensaver_cmd` shopt was implemented specifically because I wrote a program called `whoa` that has some terminal screensavers, and I wanted some way to just have `shed` run it after a certain period of time. It worked as intended, and I didn't have a reason to come back to it, so it just kind of sat untouched for a while.

Recently though I took another look at the feature and realized the implementation is kind of hamfisted. The idea is "automatically run this command after a certain condition is met", and we technically already have an entire subsystem for that - the `autocmd` builtin. So the change here is pretty clear: instead of having an entire shopt dedicated to scheduling a single command, we instead replace the `screensaver_cmd` and `screensaver_idle_time` shopts with `idle_timeout`, and then add an `on-idle-timeout` autocmd group.

Writing this wasn't actually as clean as I expected it to be. `shed` has two different pathways for command execution, one for interactive contexts (running in a terminal) and one for non-interactive contexts (running a script, etc). Autocmds typically use the non-interactive pathway, which is wrong for the `on-idle-timeout` context since it only happens when sitting on an interactive prompt. This necessitated writing an entire new pattern for `autocmd` execution, but ultimately turned out to be worth it. The `idle_timeout` shopt could now be used to schedule several different commands instead of just one, and exposing `IDLE_SECONDS` as a variable for the `autocmd` execution allows the commands to dispatch based on the total idle time. For instance, my personal timeout autocmd runs after 10 seconds, and executes different commands on different multiples of 10. After 10 seconds, it refreshes the prompt, after 600 seconds it runs my screensaver command, etc.

So great, that feature has been made pretty general, instead of being permanently pigeon-holed into the specific screensaver command use-case. So what about the other thing I mentioned above - the builtin tab completion fuzzy finder?

This one is more contentious. `shed`'s README.md actually points out the builtin fuzzy finder for both completion and history searching as one of `shed`'s selling points, but I recently implemented something that renders it effectively obsolete. The feature takes the same basic shape as the screensaver case mentioned above. Currently, the `on-completion-start` autocmds are given the list of candidates in a variable called `$MATCHES`. The variable is read-only however. My idea was to essentially make `$MATCHES` a mutable array, so that the autocmds could post-process the collected candidates. The `$MATCHES` array that lands after the autocmds run is what would be used for completion, and if only one candidate remains at that point, the completion menu is skipped altogether.

What this effectively means is that `shed`'s completion system can now be overridden entirely by custom logic. To test this out, I decided to see if I could use `fzf` to read the candidates and select one. And it worked pretty much entirely as expected. That's very cool of course, I completely rerouted the completion UI using an external program, but this also means that `shed`'s fuzzy finder is redundant.

Technically the fuzzy finder still has a place to live as a history completion UI, but it is hard to justify keeping it around as a completion front-end now, especially when the grid UI exists as an alternative. I am currently working on implementing a similar system of mutating the `$MATCHES` array for history completion as well, which will mean that `fzf` can be used there as well, which makes our fuzzy finder well and truly obsolete.

It is pretty interesting how features like this can go from "this is a load bearing part of the program's identity" to "this is useless bloat that we can cut out of the codebase entirely" just by virtue of generalized extensibility overshadowing their specialized use case.

Regardless, I think I am going to keep the fuzzy finder for history searching, but redo the look of it to match the grid completer, and then remove the fuzzy completer style for tab completion entirely.
