---
published: true
layout: post
title: Notes on efficient packing of OpenType layout
---

In terms of programming style, I'm a hacker. I code first and do the design as I'm going along. You can get pretty far with this. You can also get into terrible messes, and end up with restrictive architecture that takes ages to unpick afterwards. So sometimes you need to think first and program later. Currently I'm working on code in Rust to emit binary OpenType Layout tables, and this is definitely one of those think-first times. So here I am, thinking first.

My Rust font library uses serde to handle binary serialization and deserialization, and it's very neat. You can break up the subtables into individual structs and write (or derive) serialization code for them independently. Good for testing, good for hacking. Not good for OpenType Layout. Why not?

* Thinking about each subtable independently means you lose the ability to make global optimisations. For example, coverage tables (and indeed most subtables) are referred to by offset, meaning you can share coverage tables between lookups. Gathering all the coverage tables separately and handing out offsets to unique coverage sets is more efficient than letting each subtable handle its own serialization.

* Overflow is a problem. In fact, the problem of overflow in the Python writer is what is forcing me to sit down and try to think of a better architecture. 

To summarize the overflow problem: 

Offsets in the layout tables - in the Lookup List Table which lives at the root of the GSUB/GPOS table, in the Lookup (which wraps a bunch of subtables), and in the subtables themselves - are 16 bits. You can use 32 bit offsets in an Extension Lookup subtable, but you have to start with 16 bit offsets. The implications of this are:

* You can put the Lookups wherever you like - you can intermingle them with the subtables, and gain a bit more "stretch" - but obviously the furthest Lookup can only be 65535 bytes away from the Lookup List Table. (A Lookup is 8 bytes + 2 * the number of subtables).

* You can share lookups! You can and should share other subtables, such as coverage tables.

* Then the maximum distance between a subtable and any dependent subtable (notably the Coverage table, so we'll use that as our example) is 65535. So you can't put coverage tables *miles* away from the lookups.

* Offsets are unsigned, so shared coverage tables have to be after everything that references them.

* Clearly any lookup that has subtables can't be bigger than 64k. So you need a way to split big lookups.

* Extension Lookups help, but not as much as you'd want. They wrap a subtable at a time. So you *still* need all your Lookup's subtable offsets to be 16 bits.

* Also: wrapping a lookup with *n* subtables into a set of *n* Extension Lookups increases the number of lookups by *n-1*, which puts pressure on the Lookup List Table on the first page. (I have a font which is so congested that every time you try to fix an overflow by promoting a lookup to a set of extension lookups, this increases the size of the Lookup List Table and causes another subtable to fall off the end of the 64k page and overflow, in a kind of ripple effect. This takes Python *ages* to resolve, because Python tries a build, waits to see if an overflow happens, tries a resolution, and then loops, instead of keeping track of the max offset. So there's a hint about what a better algorithm might look like.)

* Even if you wrap a lookup's subtable into an Extension Lookup, any offset it contains still has to be (a) positive and (b) 16 bit. So you still have to put it and its coverage tables (including any shared ones) within a 64k page.

* But also, when you're promoting a subtable to an Extension Lookup (and especially when moving it beyond the first 64k offset boundary), you should probably also consider which other subtables you should move as well to get the benefit of shared coverage tables etc.

So this is kind of a [0-1 knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem), but you also have some things you want to keep together in the same knapsack, and you can break items in the knapsack in half if you need to.

Perhaps thinking about it in terms of splitting the lookups into 64k pages and trying to build each page separately is a useful way to start.

But now I sit down to think of it, this is a *really* hard problem. I'm glad to see that the [Harfbuzz repacker](https://github.com/harfbuzz/harfbuzz/blob/master/src/hb-repacker.hh) is being written by better programmers than me, and to be honest, I think what I'm going to do is just sit tight and wait until they've solved it...

