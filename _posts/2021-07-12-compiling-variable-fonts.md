---
published: true
layout: post
title: What happens when you compile a variable font
---

(with the open source tool chain.)

How do you build a variable font using free software? When everything goes right, it's very simple:

1. Type `fontmake -g YourFile.glyphs -o variable` or `fontmake -m YourFile.designspace -o variable` (if you have a designspace file).
2. Magic happens.
3. You get a variable font in the `variable/` directory.

Unfortunately, if things don't go right, then step 2 is not an adequate description. Let's look in a bit more detail about how the process operates.

`fontmake` is a simple wrapper around an ecosystem of Python libraries, each of which plays a part in the creation of a variable font. Because of this, it's very rare that a problem compiling a font is due to a problem with *fontmake*; but I often find it quite tricky to work out precisely which component has the problem. So let's go through the process from end to end, and look at the different pieces involved.

## Convert to designspace and UFO

If you have designspace + UFO sources, you skip this step and move onto the next one.

If you have a Glyphs format font source, the first thing we do is convert it into designspace + UFO. This is done through the [`glyphsLib`](https://github.com/googlefonts/glyphsLib) library. This library has two distinct components: a Glyphs file parser, and a designspace + UFO builder.

The Glyphs file parser is reasonably simple. The builder component, on the other hand, is absolute madness and I try to avoid it as much as possible. It is responsible for handling all the Glyphs cleverness - feature replacement, brace and bracket layers, axis mapping, anchor propagation (I don't really understand what that is, to be honest) - and turning it into a set of UFO files and a designspace file.

These end up in the `master_ufo/` directory. So if things go wrong fairly early on in the fontmake process, you can look in there and see if (a) the conversion succeeded at all, and (b) the results look like what you would expect them to. You can also run the process manually by using the `glyphs2ufo` command line tool. If that didn't work out well, there's a problem at the `glyphsLib` stage. If not, we go on to...

## UFO to FT

Now we have a designspace file and a set of UFOs. What we're going to do next is convert each of these UFOs in turn into a TTF file. This is done by the [`ufo2ft`](https://github.com/googlefonts/ufo2ft) library - `ft` in this instance being `fontTools`, the Python library which actually reads and writes the TTFs. `ufo2ft` takes each UFO file and goes through the following steps:

### Pre-processing

Preprocessing steps are those which are done *on the UFO source*. It's done in memory, so it's not written to the file, but it's conceptually done on the source font level.

The very first thing that happens, bizarrely, is that if you don't have a `.notdef` glyph, `ufo2ft` makes one for you. After that, fontmake will say something like `Pre-processing glyphs`.

Now, if you have a Glyphs file, `glyphsLib` will have added a custom preprocessing step here to handle the fact that Glyphs allows [outside open corners](https://github.com/googlefonts/fontmake/issues/288), so we have a filter which removes them.

Next, for each glyph, we convert the Bézier curves from cubic (on-curve, handle, handle, on-curve) to quadratic (on-curve, handle, on-curve).

TrueType fonts are based on quadratic curves, but the design tools we use are largely based on cubic curves. This means we have to reduce the order of the Béziers, and this gets problematic.

First, lowering curve order is not an exact operation; the best way to do it is to try it, check whether it was accurate enough according to a given tolerance of units along the path, and if not, split the curve in two and try again. The problem when doing this with a variable font is that the curves in different masters are, naturally, going to be different. So if you have a curve on one master which was "accurate enough" first time around and didn't require splitting, but trying the same operation on *another* master - perhaps the curve was wider or tighter, or whatever - *didn't* get it accurate enough and you had to split the curve, suddenly you have three nodes on one master and five nodes on another master, and that means your font won't interpolate.

So ufo2ft uses another library called [`cu2qu`](https://github.com/googlefonts/cu2qu) to perform the conversion across all UFO files at the same time, trying different levels of tolerance until the curves have the same number of split points across all the masters. Now we have interpolatable, quadratic UFO files.

Next, glyphs which have both paths *and* components get their components decomposed into paths, because you can't have mixed glyphs in TTF files - it's got to be one or the other. (Glyphs which are purely made up of components are just fine.)

Finally, and if the `--flatten-components` flag (highly recommended) is given to `fontmake`, then nested components (components which call other components) are decomposed into a single level.

Now we have a UFO file which is ready to become a TTF file, so it's over to...

### Building OpenType tables

We're going take the outlines from our UFO file and create a set of TrueType tables from them. For the default master, we build a complete TTF file, but for everything else, we just build the tables that we need: `glyf` (TrueType outlines), `head` (font metadata), `hmtx` (horizontal metrics), `loca` (index to the outline data), `maxp` (statistics) and `vmtx` (vertical metrics). This is all handled by the [outline compiler](https://github.com/googlefonts/ufo2ft/blob/main/Lib/ufo2ft/outlineCompiler.py), which is a bit of a misnomer because it doesn't just handle the outlines but things like name tables and indeed everything else that makes up a binary font file.

It's at this stage that the OpenType layout magic also happens. (Yes, I'm aware that "magic happens" is not an adequate description, but we're going to get there.)

This is very important to understand. OpenType layout rules are stored in *two ways* in a font file. There are the explicit rules, such as ligatures (`sub f i by fi;`); even if these rules are generated automatically by your font editor, they generally end up stored explicitly and by this stage will have made their way into the `features.fea` file of the UFO source.

But there are also implicit rules. Kerning tables are also just a set of OpenType layout rules (things like `pos A V -120;`). That's literally all they are. But this fact is hidden from you by the interface of your font editor (thankfully). Anchor positioning and mark attachment is similar; you just drag an anchor around in the editor, but at some stage, this has to become `markClass acute <anchor 150 -10> @TOP_MARKS` or something similar. The problem is that the source files don't store these rules explicitly as part of the layout rules - just the kerning pairs or the anchor positions. 

So ufo2ft has a set of [feature writers](https://github.com/googlefonts/ufo2ft/tree/main/Lib/ufo2ft/featureWriters) which convert the implicit rules in your font to explicit rules in a feature file: kern features (and `dist` for Indic scripts), mark-to-base and mark-to-mark features, as well as a GDEF table writer which tries to work out the right OpenType categories (mark or base) for each glyph.

Now we have some TTF files, it's time for...

### Post-processing

All that happens in the post-processing step is that glyphs get renamed to their production name: anything outside the [Adobe glyph list](https://github.com/adobe-type-tools/agl-aglfn/blob/master/aglfn.txt) is renamed to `uniXXXX` according to the hex value of its Unicode codepoint.

At this point, we have:

* A TTF file representing the default master, which is complete.
* A set of TTF files for the other masters, with just outlines, metrics and layout information.

Now we do the...

## Merge

Oh, how I hate the merge. It was a really good idea at the time, and it helped us get variable fonts out there quickly, but...

We now have to turn these multiple, per-master TTF files into a single variable font. This is done by `fontTools.varLib`. This takes the designspace file, which now (in memory) refers to the TTF files we've generated, and the TTFs, and does the following:

* Using the designspace file to find the axes and instances, creates an `fvar` table in the default master TTF.
* Similarly adds `STAT` and `avar` tables.
* Takes the different metrics tables - `hhea`, `vhea`, etc. - from each of the master-specific TTFs, and turns the deltas between each of these and the default master into `MVAR`, `HVAR`, `VVAR` and other tables.
* Turns the differences in the kerning and anchor rules (in the `GPOS` table) between the default master and each other master into a variation store in the `GDEF` table. This assume that all the rules are compatible, and each master has the same set of anchors and same set of kern rules. (Much can go wrong here.)
* Turns the differences between the outlines into a `gvar` table.
* Turns the differences in hinting instructions (assuming they are compatible) into a `cvar` table.
* Uses the feature substitution rules in the designspace file to create an `rvrn` feature in the GSUB table.

If things go wrong *after* `Writing OpenType tables`, it's probably due to the merge, and probably due to some kind of incompatibility in your layout between masters. But as half of the layout was implicit and generated by `ufo2ft`, it's often hard to tell whether this was your fault, `ufo2ft`'s fault, or `fonttools.varLib`'s fault. I recently added some [better diagnostics](https://github.com/fonttools/fonttools/pull/2223) to help understand why the merge fails, but it's still deep voodoo.

But if things don't go wrong, then we're done: `glyphsLib`, `ufo2ft`, `cu2qu`, and `fontTools`, all orchestrated by `fontmake`, have combined to make a variable font.

Now, hopefully, you know which bit does what.
