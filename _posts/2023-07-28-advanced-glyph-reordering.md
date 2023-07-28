---
published: true
layout: post
---

Yesterday I was re-engineering a font to solve some reported issues, and I ended up implementing an OpenType Layout technique which deserves to be better known. But first you're going to have sit through a rant.

I understand that programming layout rules is hard, especially for less common scripts, and I get that designers aren't programmers. And I suppose it's a good thing that designers solve problems in ways that they can understand, using tools and techniques that they're familiar with. But sometimes this can lead to storing up problems for later.

For example, despite the best efforts of the Universal Shaping Engine, some scripts can end up with the input glyphs not in visual order. I was working on Khojki, and even though you type `<BASE> nukta shadda` you want the nukta to appear on top of the shadda. One way to deal with this is to form a ligature: `sub nukta-khoj shadda-khoj by shadda_nukta-khoj;`.

This is easy. Everyone knows how ligatures work; you can see them in the glyph editor; you can edit and reposition them. But wait. Now you have another glyph in your font. If you have lots of above-marks to rearrange, you end up with *lots* of additional glyphs in your font. And as I've said [before](https://simoncozens.github.io/mark-to-base/), more glyphs means more maintenance. The right approach here is to swap the glyphs around and use mark-to-mark positioning.

Why? Well, Noto Serif Khojki has an alternate form of the nukta, which is inverted. If we don't the ligature route, we do `feature ss02 { sub nukta-khoj by nukta-khoj.alt; } ss02;` and we're done. If nukta can be part of a ligature, then *every* ligature glyph involving a nukta needs an alternate form, we need a bunch more glyphs in the font, we need a bunch more substitution rules, and it's all a nightmare. Swapping the glyphs is harder up front but it pays dividends later.

So let's think about how to do a *basic* reordering of two marks. The idea is that we will have a base glyph, potentially some marks which we don't care about (below marks, for instance), and two marks which we do care about and want to swap. So in `ccmp` or similar, we do this:

```fea
feature ccmp {

    lookup SwapNuktaShadda {
        lookupflag UseMarkFilteringSet [nukta-khoj shadda-khoj];
        sub @BASES nukta-khoj' lookup NuktaToShadda shadda-khoj' lookup ShaddaToNukta;
    } SwapNuktaShadda;

} ccmp;
```

> If you're dealing with any scripts with complex layout requirements, `UseMarkFilteringSet` is your new friend and you should get really familiar with it and use it practically everywhere. It focuses the shaping engine on a particular set of marks and ignores others, which allows you to do stuff to a particular glyph string without needing to worry about the presence or absence of other marks. I deal with many bugs where the presence of a below mark blocks a substitution of above marks or vice versa because the engineer didn't know about `UseMarkFilteringSet`.

And somewhere in a prefix we define the two lookups:

```fea
lookup NuktaToShadda { sub nukta-khoj by shadda-khoj; } NuktaToShadda;
lookup ShaddaToNukta { sub shadda-khoj by nukta-khoj; } ShaddaToNukta;
```

When you get a bit more familiar with the technique, you can combine things like this:

```fea
lookup DoNuktaShaddaSwap {
    sub nukta-khoj by shadda-khoj; sub shadda-khoj by nukta-khoj;
} DoNuktaShaddaSwap;

feature ccmp {
    lookup SwapNuktaShadda {
        lookupflag UseMarkFilteringSet [nukta-khoj shadda-khoj];
        sub @BASES nukta-khoj' lookup DoNuktaShaddaSwap shadda-khoj' lookup DoNuktaShaddaSwap;
    } SwapNuktaShadda;
} ccmp;
```

OK, so that's the basic swap. Do that, use mark-to-mark attachment, and you can cut down on the number of ligatures you use. Now onto the technique I found yesterday.

Another issue with Khojki is that as well as taking potentially a nukta and potentially a shadda, a base can also take a vowel sign. The vowel sign is encoded *after* the above marks, but needs to be displayed before them with the other marks on top. So we have a sequence like `tta-khoj nukta-khoj shadda-khoj eMatra-khoj` and we *want* a sequence like `tta-khoj eMatra-khoj nukta-khoj shadda-khoj`.

Now the ligature approach starts to get really horrible:

```fea
sub nukta-khoj shadda-khoj eMatra-khoj by eMatra_nukta_shadda-khoj;
sub nukta-khoj eMatra-khoj by eMatra_nukta-khoj;
sub shadda-khoj eMatra-khoj by eMatra_shadda-khoj;
sub nukta-khoj shadda-khoj by nukta_shadda-khoj;
```

Three extra glyphs per vowel; and to support our alternate nukta forms, two more extra glyphs on top of that. And that's just for one vowel sign; there are seven of them with this behaviour. If the only tool in your box is a ligature, you're now adding thirty-five utility glyphs to this font, and I am wading through all these glyphs at a later date cursing you.

But I admit that the reordering here is tricky, and it had me scratching my head for a little while. What we want to do is ignore the other optional marks, reach out and grab the vowel sign and pull it next to the base. Then we reorder our vowel signs, then we are done and don't need any ligatures. But how do we do that? The basic idea goes like this. (I'm just going to show it for the `eMatra-khoj` glyph; other vowels are analogous):

```fea
feature ccmp {
    lookup GrabMatra {
        lookupflag UseMarkFilteringSet [eMatra-khoj];
        sub @BASES' lookup AddE eMatra-khoj' lookup DeleteMatra;
    } GrabMatra;
} ccmp;
```

We'll look at how the add/delete lookups are defined in a moment, but let's just check our understanding of what this is doing. Because it *looks* like it adds a glyph and deletes the glyph, and surely that does absolutely nothing, right? The trick to understanding this is what I call the "paper tape" model of OpenType layout and you can read more about it [here](https://simoncozens.github.io/fonts-and-layout//features.html#lookup-application).

In this case, the "read head" of our tape machine marches along the glyph stream, looking for two things that match - a base and an `eMatra-khoj` glyph - *while ignoring all marks apart from `eMatra-khoj`*. When it finds those two things, it will call `AddE` at the location of the base, and call `DeleteMatra` at the location of the `eMatra-khoj` - and because we are ignoring marks apart from `eMatra-khoj`, these two locations can be quite a number of glyphs apart. This is what we want:

<img src="https://github.com/simoncozens/simoncozens.github.io/raw/master/images/advanced-reordering-1.png">

`AddE` will be defined like this:

```fea
lookup AddE {
    sub ka-khoj by ka-khoj eMatra-khoj;
    sub kha-khoj by kha-khoj eMatra-khoj;
    sub ga-khoj by ga-khoj eMatra-khoj;
    # ...
} AddE;
```

This is boring and repetitive but at least it's easy to understand, and hopefully you can see that because we called `AddE` at the location of the base, the new `eMatra-khoj` will be added *right next to the base*, which is where we want it.

Now for the tricky bit. When I came to implement `DeleteMatra`, the first thing I tried was:

```fea
lookup DeleteMatra {
    sub eMatra-khoj by NULL;
} DeleteMatra;
```

This is a slightly obscure technique for deleting a glyph, which was only "discovered" fairly recently; it's not documented in the specification and was found to be implemented in the Uniscribe shaper and then picked up by other shaping engines for compatibility purposes.

But this didn't work. If you can't see why not, stop for a moment and try to work it out. The "paper tape" model will help; in fact, you won't be able to get it without this understanding of how OpenType works.

> ... Did you see it? ...

The key is that when we did `AddE`, we *added an extra glyph to the glyph stream*, but this does *not* change the position of the "read head" which is trying to apply the `DeleteMatra` lookup. After we have added the extra glyph, the situation now looks like this:

<img src="https://github.com/simoncozens/simoncozens.github.io/raw/master/images/advanced-reordering-2.png">

The read head of the `DeleteMatra` application now sits at the position of the glyph *we just added*. `DeleteMatra` looks at the glyph it points to, finds that it's an `eMatra-khoj`, and happily deletes it, leaving us exactly back where we started; oh, the embarrassment. (If you use [Crowbar](http://corvelsoftware.co.uk/crowbar/) for OpenType debugging - and you *should*, if you're doing this kind of stuff - you can see it happening step by step, which really, really helps understanding what's going on.)

Now it happens to be the case that, in the underlying OpenType format, you can actually specify which order the lookups in a contextual substitution apply. If we were able to say "Call `DeleteMatra` first at the second location, and then call `AddE` afterwards at the first location" - and we can, in the binary format - then everything would be fine. The change in the position of the glyph stream would happen after we are done deleting things, so it wouldn't matter. But unfortunately we *can't* express the order of application using the Adobe feature syntax. (I opened [an issue](https://github.com/adobe-type-tools/afdko/issues/1167) about this three years ago, but nothing happened.) So we need to get clever.

What we need to do instead is have the `DeleteMatra` lookup "reach ahead" to the *next* glyph in the glyph stream, the one *after* we just added. How on earth do we do that? Well, the way to ask OpenType to match glyphs and perform a lookup at a certain position in the glyph stream is to use a contextual substitution. So this means we end up using a *second level* of contextual substitutions, like this:

```fea
lookup ReallyDeleteMatra { sub eMatra-khoj by NULL; } ReallyDeleteMatra;

lookup DeleteMatra {
    lookupflag UseMarkFilteringSet [eMatra-khoj];
    sub eMatra-khoj' # This is the new glyph we just added
        eMatra-khoj' lookup ReallyDeleteMatra; # This is the old one we want to delete
} DeleteMatra;

feature ccmp {
    lookup GrabMatra {
        lookupflag UseMarkFilteringSet [eMatra-khoj];
        sub @BASES' lookup AddE eMatra-khoj' lookup DeleteMatra;
    } GrabMatra;
} ccmp;
```

Now this is what happens:

<img src="https://github.com/simoncozens/simoncozens.github.io/raw/master/images/advanced-reordering-3.png">

And now our glyphs are just the way we want them.

I admit that this is complicated *(All right, sure, it's absolute bloody witchcraft)* but once it's done, it's done, and all the glyphs are now in the right order, which is clearly the Right Thing, and the font is smaller and more maintainable and much easier to reason about. And I encourage you to do the Right Thing.

It takes a bit of thought up front, but you'll thank me in the end.
