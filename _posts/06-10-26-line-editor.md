---
layout: post
title: Shed's Line Editor
date: 2026-06-09
---

  I use `vim` as my main text editor. In every shell I've ever used, I always used the integrated `vi` keybindings. Words cannot describe the pure unfettered agony of attempting to use some vi motion/verb in the shell's line editor and finding out that the line editor had the brazen audacity to *not implement it*. And then call itself a `vi` mode anyway. I ran into this problem many times during my testing using `rustyline`, and realized that if I was writing a shell anyway, I had the power to create a line editor where this problem *does not exist*. A line editor with a `vi` mode that matches `vim` so closely it could be considered an integrated emulator of the program.

  I mainly wanted to see for myself why this hadn't already been done before. Was it limitations caused by the prompt being embedded in the terminal directly? Technical debt preventing proper extension? Incompatibilities with common line editor keybindings? Just plain laziness perhaps? So I got to work on `shed`'s line editor, trying to find exactly where the idea of creating a `vim` emulator as a line editor would fall apart.

  The `ShedLine` struct is the actual "line editor" object; it wraps the `Completer`, the command `History`, and the `LineBuf`. The interesting field is `Box<dyn EditMode>`: a trait that takes `KeyEvent`s and returns `EditCmd`s, which the `LineBuf` then applies to the buffer. Implementors of `EditMode` are modeled after vim's editor modes; each one parses keystrokes the way the corresponding vim mode does.

  As for the actual buffer itself, the `LineBuf` struct was created. It is currently the single largest struct in the entire codebase. Not surprising, because that's where all of the business logic of our `vim` emulator lives. Originally, when `LineBuf` was first written, it simply wrapped a `String` and performed direct mutations on that `String`, according to the `EditCmd`s that it received. This worked for a long time, but the cracks in this model began showing once `vim`'s more line-oriented features started coming into play.

  Motion/verb combinations that spanned multiple lines, such as `2dj`, were very hard to nail down airtight logic for. The best that could really be done was scanning the buffer for newlines, and using the spans in between the newlines to represent the lines. But this ended up being very flimsy. For instance, was the span used supposed to be an open or closed range? How would the line boundary interact with cursor movements? The system constantly fell apart under any kind of stress, and needed special cases everywhere, like here we need to subtract one from the end of the line's span, but over here we don't, and over here we need to take the line's span but add one to the start. It was a huge mess.

  I imagine this is probably the point where the "vim emulator" idea was supposed to fall apart, and likely why most shell `vi` mode implementations are so limited in scope. However I am stubborn and wanted to be able to do ridiculous things like type `:%normal!g?$` and have it actually work. Will that ever help me write a shell command? Probably not, and I don't really care. Science is not about why, it's about why not.

  So the path was straightforward: in order to get my `vim` emulator, we would have to drop `String` as the text representation, and write our own data structure. The result was this:
  * `Lines(Vec<Line>)`
  * `Line(Vec<Grapheme>)`
  * `Grapheme(SmallVec<[char;4]>)`

  Essentially, a ***3-dimensional vector*** of `char`s. The reason I went with a 3D vector instead of a 2D vector, was so that all of the `chars` held by multi-codepoint clusters could be *stacked on top of each other* instead of written as a linear sequence in the line. The result is that each individual index of `Line`'s inner vector always represents a single Unicode character, and higher level logic never ever ever has to care about things like `char` boundaries or zero-width joiners or any of that crap.

  This change made line-wise operations trivial, but actually made char-wise operations that span multiple lines quite complex. The main difference in the complexity this time around was that the logic for extracting a range, and then joining the head and tail of the resulting buffers, and other similar operations could be packed into helper methods on the `Lines` and `Line` structs. The same can't be said about the flat `String` that was being used before.

  This resulted in a text editor data structure that welcomed extension with open arms. After implementing support for almost all of normal mode's operators, visual mode, registers, keymaps, and even macro recording, the logic of the editor remains robust. A rather interesting side effect of writing a line editor that is a `vim` emulator first and foremost, is that later implementation of the standard `emacs`-style keybindings that other shells use was actually a trivial affair. It was as simple as writing an `EditMode` struct that behaved similarly to `Insert` mode, with some `emacs`-related operations like killring cycling. This development probably stands in opposition to how most line editors were probably written, where the editing logic for the buffer was written for the simpler `emacs`-style keybindings, with the `vi`-style keybindings being bolted on later.

  I won't say that the line editor's current state is a perfect solution. The `Lines` struct is actually very large (each `Grapheme` reserves space on the stack for 4 `chars`), and could probably do with some kind of optimization. There is also room for optimizing the syntax highlighter on larger buffers. But even so, in its current state, `shed`'s line editor has actually exceeded my original expectations. I can easily say that I am more productive with it than with any other shell I've ever used.
