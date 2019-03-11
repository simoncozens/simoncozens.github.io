---
published: true
title: More on newbreak
layout: post
---

I had another mess around with [newbreak](https://github.com/simoncozens/newbreak) on Friday; this is my hyphenation-and-justification engine. The main idea behind newbreak is that text can be stretchy as well as space, something we need for virtual fonts - perhaps for effect in display situations using Latin script, but something that's really needed for non-Latin scripts.

Gor Jihanian contacted me with a great example: Armenian. He asked if it would be possible to extend the engine to stretch the glyphs of Armenian poetry, which required adding support for a few more things.

* Hard line breaks

Previously `dombreak` grabbed the text, turned all the whitespace into glue nodes and handed them to `newbreak`. The problem with this is that when you're setting something like poetry, you want to keep the line breaks. So I added a thing which turned hard linebreaks into a very stretchy piece of glue and a penalty, just like TeX. Except that you don't always want the stretchy glue because...

* Full justification

The whole point of this particular exercise was to render the text flush, with full justification, expanding the glyphs in order to get the lines to line up:

![Screenshot 2019-03-10 at 20.06.24.png](https://github.com/simoncozens/simoncozens.github.io/raw/master/_posts/Screenshot%202019-03-10%20at%2020.06.24.png)

All it takes to do this is to *not* add the very stretchy glue before hard breaks and paragraph endings, so that was easy too.

* Arbitrary variable font axes

This was the slightly tougher one. Jor's font used the [GEXT](https://github.com/jmsole/gext-demos) (glyph extension) axis, rather than the usual width axis. I had to edit the function in `dombreak` which actually does the stretching and shrinking of glyphs to allow the variation axis to be specified.

* Automatic stretch/shrink detection

This was a key one. Previously I had assumed that all glyphs in the script stretch to the same amount, but Jor's font only stretched certain glyphs. That means that the `stretch` parameter of a node was not merely a fixed multiple of the node width, but needs to vary depending on the word - which, frankly, is the Right Thing To Do in any case. Not all glyphs will or should stretch equally. So I added the option to automatically determine how far the font will allow a bit of text to stretch or shrink by attempting to set it to infinite points and zero points respectively and then measuring how far it actually did go.

The final result was quite pleasant:

![armenian.gif](https://github.com/simoncozens/simoncozens.github.io/raw/master/_posts/armenian.gif)

Jor raised a few more interesting questions for future development of `newbreak` and `dombreak`:

* Can we assign a priority to stretch certain glyphs more than others?

Yes: compute your stretch parameter carefully, adding up the stretchiness you would like to give to each glyph in the word.

* Can we stretch certain glyphs *first* and then others if more stretch is needed?

Mmm. Not sure.

* Can we penalise having too many glyphs stretched in a line?

I suppose this already happens because the badness computation penalizes stretching. But maybe we could penalise stretching glyphs *more* than we penalise stretching whitespace; at the moment any stretch is a stretch, and no distinction is made based on what we are stretching.

* Can we penalise having glyphs stretched in a position directly below a stretched glyph on the previous line?

Urgh. In theory, but the computation starts to get very difficult.

* Can we stop certain glyphs from stretching in certain cases (i.e. one-letter words, or glyphs in final position)?

Sure, just manually supply the correct stretch parameters. In the case of one-letter words, make a node with a stretch of zero. In the case of a word with a glyph in final position which is normally stretchable, split it into two nodes and make the final node have a stretch of zero.

As an implementation note, adding hard breaks (which is implemented by massive negative penalties at a particular point in the line) meant I had to dismantle one of my optimizations. I had previously kept a running total of demerits for each possible set of breakpoints, and if the algorithm went down the tree and acquired more demerits than the current winner, it would short-circuit the recursion because things were only going to get worse from there. But if you have negative penalties at certain nodes, you can actually start picking up negative demerits and *improving* your situation. So you have to check everything. I compromised on short-circuiting the case where you've got more demerits than the winner and you know there are no possible negative penalties coming up, but that's still a bit annoying. The algorithm is already pretty slow - by design, because I've written it for simplicity and extensibility rather than for best algorithmic complexity.

I had started to wonder if I could simply make a two-line change to the Knuth-Plass algorithm to add stretch and shrink to the running totals on box nodes and well as glue nodes, because it's obviously going to be *loads* more efficient. But Jor's questions above made me realise it's better to keep playing with a new implementation for now, to more easily allow this kind of extensibility.

Maybe once we have worked through all the things we would like to do with variable font justification, we can go back and make an extremely efficient engine to do it, possibly based on the Knuth-Plass model. But you know what? I still find the best way to optimize code is to leave it for six months, and let computers get faster in the meantime.

Next I will start looking at using `newbreak` for Arabic kashida justification, which is what I have been working towards with all this. Safar has sent me some example fonts. But I have dayjob stuff to do this week, so I shall try my best to resist the siren call of messing with text layout until that is done.
