---
published: true
layout: post
---

This update just went out to [GitHub Sponsors](https://github.com/sponsors/simoncozens).

## Justified and Ancient

January started back with a visit to the University of Reading Department of Typography, where I met with Titus Nemeth of the TypoArabic project. He's been thinking about how to improve Arabic computer typesetting, and I'm interested in the problem of computer justification. We talked about how Arabic lines *ought* to be justified, and what computers can do to support them. We determined that the way Arabic is justified in practice is: first, through the use of "justification alternates" - swapping out glyphs for swash or ligature forms which improve the line fit; then through adjustment of spacing; then through insertion of kashidas.

The OpenType specification defines a [JSTF table](https://docs.microsoft.com/en-us/typography/opentype/spec/jstf) inside the font to support advanced justification, and it *sort of* looks like it will do all the things we want it to, but there's a problem: nobody has ever implemented it - not on the font editor side *or* on the layout engine side. There are no JSTF fonts, and nothing that can read them. **Challenge accepted!**

So after initially hand-hacking an OpenType font(!) to add a JSTF table, I ended the day by writing a [utility](https://gist.github.com/simoncozens/cad7640a37265b087b35c2c59a510f9f) to add simple JSTF tables to arbitrary fonts, as well as an extension to SILE to read the table out of the font and provide justification alternates. The code was horrible and only barely worked, but it was a start.

Part of the problem is that SILE's line-breaking engine (stolen wholesale from TeX) only deals with adjusting spaces. My hacks to it to try to make it support justification alternates didn't really work well. We needed a new justification engine. Thankfully, I had an [experimental justification engine](https://github.com/simoncozens/newbreak) hanging around just for this purpose. I rewrote major chunks of it to completely support both justification alternates *and* glyph extension *and* space adjustment, *and* the ability to configure how those three strategies should interact.

This was all written in JavaScript as a prototyping environment, which produced some interesting examples, first for [Arabic](https://twitter.com/simoncozens/status/1217736872992104448) and then, for the sheer fun of it, [of the Gutenberg Bible](https://twitter.com/simoncozens/status/1217862777559289858).
 
The next steps for this project are to port this new engine to C (and possibly also to Lua) so that it can be used in SILE (and more generally in other people's projects, should they want it), and then:

* Fully support reading JSTF tables in SILE
* Improve the JSTF table builder script so that it supports more than simple extension substitutions
* Finally, suggest some feedback to the OpenType spec developers to improve the JSTF table now that there is a real-world implementation at last

## Fonts and Layouts Book

I've done quite a bit more on chapter 7 of the [book](, which is at time of writing clocking in at 5,500 words. (The whole book is currently 40,000 words.) This chapter is all about practical examples of supporting complex scripts in OpenType, and to be totally honest, I'm learning a lot as I go along!

I've been able to come up with some really good examples which don't just spoon-feed the reader solutions but which encourage them to think about how to use the tools available in OpenType Layout to solve their problems. I think I've also been able to demonstrate some good techniques for developing complex Arabic fonts. I need to show similar tips and tricks for Devanagari, which is an area I actually know a lot less about.

I'm also currently digging in to how the Universal Shaping Engine works, which is extremely important for people developing for minority scripts - because the USE relies only on the Unicode data and not any script-specific processing on the shaper, once a script is correctly encoded in Unicode, it's automatically supported by shapers, which reduces the barrier to entry dramatically. But material on how it all works is quite sparse ([one web page](https://docs.microsoft.com/en-us/typography/script-development/use) and [one conference presentation](http://tiro.com/John/Universal_Shaping_Engine_TYPOLabs.pdf)), so there's a need for something designer-friendly to explain how to write fonts for it.

As I write and work out how best to explain things, I also come up with small utilities and bits of code to help me, some of which get released and fed back. I improved the [Unicopedia](https://github.com/tonton-pixel/unicopedia-plus/pull/4) app which displays Unicode Character Database data, have produced a number of [test fonts](https://github.com/simoncozens/test-fonts) (some of which have made their way into the [Unicode.org text renderer test suite](https://github.com/unicode-org/text-rendering-tests/commit/9598b4357944acaeebf1dfbf2992d01a7e0084da), and I'm working on a Python library interface to the Unicode Character Database.

## SILE 0.10.0

Finally, after a year, @calebmaclennan and I (mostly Caleb, to be fair) released a new version of SILE. This one mainly has internal refactorings, which will stand us in good stead for future development. The build and test systems have been massively overhauled, and SILE will now download the Lua modules it needs on installation. Caleb has also been working on packaging SILE into a Docker container. More details are on the [web site](https://sile-typesetter.org).

Thanks again for your support this month.
