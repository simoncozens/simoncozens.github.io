---
published: true
layout: post
title: FontTools Serialization - a note to self
---

I've previously mentioned a problem in OpenType called <a href="https://simoncozens.github.io/layout-packing/">table packing</a>; it's the problem of assigning byte offsets to link the various subtables inside a GPOS/GSUB table so that no offset goes over the maximum limit of 64k. It's a very hard problem algorithmically, but it's also a problem for me personally in my workflow.

The workflow problem is this: as soon as fontTools finds that an offset in a layout
table overflows, it either breaks it down into smaller tables, or promotes it to
an extension subtable, or moves it somewhere else in the graph, and then it tries
again. Any of these overflow resolution strategies can have a ripple effect on
other subtables, causing other overflows, so it has to try again *repeatedly*.
This can turn a font build from a few seconds into one which takes five minutes.
Indeed, the fonts that I'm currently working on take five minutes to build, the
vast majority of which is spent in overflow resolution.

And this is a workflow problem because of attention spans. Let's say I'm fixing
a layout issue: I find a sequence which is not right, I write a fix in the FEZ
code, and I build the font. If it takes a couple of seconds to build, I can
stay focused. Wait two seconds, see if my fix worked, and either move onto the
next problem or try again.

But if it takes five minutes to build a font, the situation is *very* different.
Either I wait for five minutes between every single experiment, which makes the
day extremely frustrating, or I fiddle with something else in the times when the
font is building; but if I do that, I'm constantly swapping my attention between
two separate tasks. Once the font is finished building, I have to go back and
remember what problem I was trying to solve in the layout and what I was doing
about it, because all that "state" has been "swapped out" of my head to work on
the other task.

I've tried a bunch of things to make overflow resolution faster. I have FEZ
variables to disable certain "heavy" parts of the layout (such as Myanmar
automatic kerning) and only build a font which doesn't overflow, if I'm testing
a part of the font which doesn't require kerning. That works nicely, but I'm
coming to the end of various projects now and so I'm having to test the font as
a whole, so that approach doesn't work. I've tried using <a href="https://www.pypy.org">PyPy</a>
for the build step, which works quite nicely, except that I've just got a new
M1 laptop and PyPy doesn't work on M1 yet.

So it's currently really frustrating to work on fonts.

(Of course there is another way around this: I could actually think about what
I'm doing, instead of randomly prodding things and seeing whether or not they work.
But that would be a *major* change in workflow.)

Now at some point the calvary is coming: the Harfbuzz team have been working on
a new repacker, and it will be integrated into fontTools. But (a) this isn't
done yet, and Harfbuzz can't actually do any of the resolution strategies we
talked about (promote to extension subtable, split into smaller tables), and (b)
I can't wait for the calvary any longer. I'm making fonts today. So I have to see
if there's anything I can do to mitigate the problem: make fontTools overflow
resolution faster. I'm thinking of porting the bit of the Harfbuzz repacker that
exists to Python, and then using fontTools' existing resolution strategies to
fill in the missing pieces.

*Unfortunately* fontTools' existing resolution strategies are part of a piece of
the library - in fact, the whole mechanism by which fontTools turns a complex
table into its binary representation - that is the most complicated and opaque
bit of the code to me. I don't quite know how to plug my object graph (from the
Harfbuzz repacker port) into whatever fontTools is doing, because I don't really
know what fontTools is doing. So I'm going to have to understand how that works first.

Sure, I understand the general idea - each OpenType table or subtable gets turned
into an `OTTableWriter` object which splits the table into an array of `items`. Each
item is either binary data or *another* `OTTableWriter` object, which somehow will get
turned into an offset. If the offsets overflow, then an `OverflowErrorRecord`
is created and then (*handwave*) the overflow gets resolved.

But that's as much as I understand right now, as I write this. So I'm going to
go through the code, try to understand how this is all put together, and how the
serialization and offset management works.

Right. I know we're going to be spending most of our time in [`otBase.py`](https://github.com/fonttools/fonttools/blob/4.28.3/Lib/fontTools/ttLib/tables/otBase.py)
The entry point is `BaseTTXConverter.compile`, and there's a [big friendly comment](https://github.com/fonttools/fonttools/blob/2d7b76aba3e66a0b47fb660a9e80a54957292079/Lib/fontTools/ttLib/tables/otBase.py#L47-L66)
there telling us the general outline. Looking at the code, the first thing we
do inside `compile` of, say, a `GSUB` table is to call `self.table.compile`,
which is a wee bit confusing. What's going on is this:

```
In [1]: self
Out[1]: <'GSUB' table at 1056b2130>

In [2]: type(self)
Out[2]: fontTools.ttLib.tables.G_S_U_B_.table_G_S_U_B_

In [3]: self.table
Out[3]: <fontTools.ttLib.tables.otTables.GSUB at 0x1056d0880>
```

Inside the top-level object representing the GSUB table itself is an `otTables.GSUB`
object (which is generated from a data-driven description of the GSUB header),
which is itself a `otBase.BaseTable` object. `BaseTable.compile` copies a bunch
of information into the `OTTableWriter` object, but critically not the subtable itself.
So the `OTTableWriter` currently has no easy way to refer to what it came from. That
seems useful for us to be aware of.

The way each "item" copies itself into the `OTTableWriter` is to [call `write` on itself](https://github.com/fonttools/fonttools/blob/2d7b76aba3e66a0b47fb660a9e80a54957292079/Lib/fontTools/ttLib/tables/otBase.py#L774).
A subtable item is an instance of `otConverters.Table`, which is a subclass of
`otConverters.Struct`. It's `otConvertors.Table.write` which [creates the sub-writer](https://github.com/fonttools/fonttools/blob/2d7b76aba3e66a0b47fb660a9e80a54957292079/Lib/fontTools/ttLib/tables/otConverters.py#L626-L631)
and recursively calls `compile` on the item, building the rest of the object graph.

At some point this graph has to get turned into actual offsets. Where does that
happen?

Well, this looks relevant:

```
	def _gatherTables(self, tables, extTables, done):
		# Convert table references in self.items tree to a flat
		# list of tables in depth-first traversal order.
```

`_gatherTables` is called by `getAllData`, which is the bit which turns the
`OTTableWriter` objects into binary, so that makes sense. First it calls
`_doneWriting`, "which removes duplicates" according to our helpful comment (so that's how that works!).

> Actually there's a bit of magic here, which took me a little while to understand:
> ```
> items[i] = item = internedTables.setdefault(item, item)
> ```
> After `_doneWriting` has been recursively called on all subitems, then the
> `OTTableWriter` objects are hashable based on their content. Hence, putting
> a writer into a dictionary and then immediately getting it back out again replaces
> it with any equivalent writer object we've already seen, or stores it into the dictionary if not. Very clever. Very opaque.

Once duplicates have been removed, `_gatherTables` flattens the graph into a list. Once we have a flat list of tables, we just run over them in turn, [allocating each one a `.pos` attribute](https://github.com/fonttools/fonttools/blob/2d7b76aba3e66a0b47fb660a9e80a54957292079/Lib/fontTools/ttLib/tables/otBase.py#L407-L414). Then when we call `getData` on each one, we use its `.pos` and the `.pos` of its children to [compute the offset](https://github.com/fonttools/fonttools/blob/2d7b76aba3e66a0b47fb660a9e80a54957292079/Lib/fontTools/ttLib/tables/otBase.py#L279-L284). And it's here that, if the offset doesn't fit, we raise an exception, create an overflow record, and try again.

OK, I have two questions at this stage.

1. What's an overflow record?
2. How do we resolve it?

But before I think about those, let's take stock of what have we learnt so far.

1. `BaseTable.compile` creates the object graph of writers by calling `write` on each item, which is an `otConvertors.Table` object, which in turn calls `compile` and around we go.
2. Each `OTTableWriter` knows its "name", but not the object where it came from.
3. `getAllData` calls `_gatherTables` to turn the graph into a flat list, then calls `getData` on the list to assign offsets.
4. If that doesn't work, *magic happens* and then the whole darned process is repeated.

Right. Let's make some overflows.

Here's some code which makes an extremely large PairPos subtable:

```python
font = TTFont("Roboto-Regular.ttf")
pos = PairPosBuilder(font, None)
x = 0
for g1 in font.getGlyphOrder()[0:120]:
    for g2 in font.getGlyphOrder()[0:120]:
        vr = buildValue({"XAdvance": (x % 1024) })
        vr2 = buildValue({"XAdvance": -( x % 1024)})
        pos.addGlyphPair(None, g1,vr2,g2,vr2)
        x += 1

lookupList = otTables.LookupList()
lookupList.Lookup = [pos.build()]
writer = OTTableWriter()
lookupList.compile(writer, font)
writer.getAllData())
```

This explodes; we can't build the lookup list. We're going to try putting this into a GPOS table later, but let's first see what's happening.

```
  File "/Users/simon/others-repos/fonttools/Lib/fontTools/ttLib/tables/otBase.py", line 567, in packUShort
    return struct.pack(">H", value)
struct.error: 'H' format requires 0 <= number <= 65535
```

Our offset was so big it couldn't fit in an unsigned short.

```
During handling of the above exception, another exception occurred:
...
  File "/Users/simon/others-repos/fonttools/Lib/fontTools/ttLib/tables/otBase.py", line 289, in getData
    raise OTLOffsetOverflowError(overflowErrorRecord)
fontTools.ttLib.tables.otBase.OTLOffsetOverflowError: (None, 'LookupIndex:', 0, 'SubTableIndex:', 0, 'ItemName:', 'PairSet', 'ItemIndex:', 91)
```

OK, we have a kind of chain of events linking us back to the PairSet item which didn't fit. There's no resolution here, because the resolution all takes place inside `BaseTTXConverter.compile`, so let's put that into a GPOS table and see what happens.

```python
font["GPOS"] = newTable("GPOS")
gpos = otTables.GPOS()
gpos.Version = 0x00010000
gpos.ScriptList = otTables.ScriptList()
gpos.ScriptList.ScriptRecord = []
gpos.FeatureList = otTables.FeatureList()
gpos.FeatureList.FeatureRecord = []
gpos.LookupList = otTables.LookupList()
gpos.LookupList.Lookup = [pos.build()]
font["GPOS"].table = gpos
font["GPOS"].compile(font)
```

If we turn on logging, we get this:

```
Attempting to fix OTLOffsetOverflowError ('GPOS', 'LookupIndex:', 0, 'SubTableIndex:', 0, 'ItemName:', 'PairSet', 'ItemIndex:', 91)
Attempting to fix OTLOffsetOverflowError ('GPOS', 'LookupIndex:', 0, 'SubTableIndex:', 0, 'ItemName:', 'PairSet', 'ItemIndex:', 91)
```

Interesting. The overflow is processed twice. Why's that, then?

This leads us to this code in `BaseTTXConverter.compile` which actually handles the overflows:

```python
    if overflowRecord.itemName is None:
        from .otTables import fixLookupOverFlows
        ok = fixLookupOverFlows(font, overflowRecord)
    else:
        from .otTables import fixSubTableOverFlows
        ok = fixSubTableOverFlows(font, overflowRecord)
    if not ok:
        # Try upgrading lookup to Extension and hope
        # that cross-lookup sharing not happening would
        # fix overflow...
        from .otTables import fixLookupOverFlows
        ok = fixLookupOverFlows(font, overflowRecord)
        if not ok:
            raise
```

What's this telling me? If we overflowed at the "top level" (i.e in the lookup list or from a Lookup to a Subtable), we call `fixLookupOverFlows`, but if we overflowed within a subtable, we call `fixSubtableOverFlows`. Aha. The first thing `fixSubtableOverFlows` does is turn sharing off and try again. Then the second time, it tries to split the subtable into multiple subtables. This is fairly easy to resolve.

The bigger problems are (a) every time something overflows we try again, and (b) lookup overflows. Well, that gives me a hint about how to speed things up: It seems to me that if the offset to lookup N is too big to fit in a ushort, the offset to lookup N+1 is going to be too big as well. In fact, all lookups from lookup N onwards are going to need handling. So rather than handling them one at a time, why not just promote the lot to extension lookups?

This is what I've implemented in [this PR](https://github.com/fonttools/fonttools/pull/2465), which reduces my build time from 5 minutes to just under a minute. And if I run that on pypy, 15 seconds. I think I can probably keep focus for 15 seconds...

