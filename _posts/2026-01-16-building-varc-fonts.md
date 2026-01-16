---
published: true
layout: post
title: "Building VARC Fonts"
---

The VARC table provides a way for fonts to define composite glyphs made up of other glyphs, where each component glyph can have its own private variation axes. This is sometimes known as "variable components" or "smart components". However...

## Terminology

...It's much easier to understand VARC if we consistently use a few key terms:


* **Public axis**: An axis that is exposed to the user in the font's `fvar` table. For example, `wght` or `wdth`.
* **Private axis**: An axis that is *not* exposed to the user, but is used internally by a component glyph. For example, a `serifStyle` axis that only affects a serif component glyph.
* **Composite glyph**: A glyph that is made up of one or more component glyphs. For example, an accented character made up of a base letter and an accent mark. A composite glyph may also have its own outlines, but we'll get to that case later.
* **Component glyph**: A glyph that is used as part of a composite glyph. Component glyphs can have their own outlines and may have public and/or private variation axes.
* **Axis values**: For the purposes of VARC, when we talk about "axis values", we mean the specific values at which a component glyph is instantiated within a composite glyph. Another way to say that is: the component’s location in its own personal designspace. For example: "*composite glyph* `T` uses *component glyph* `stem`; at `opsz=6`, `stem` is instantiated at `serifStyle=0.5`; at `opsz=144`, `stem` is instantiated at `serifStyle=1.0`. The *axis values* for `stem` are 0.5 and 1.0".


## Simple example

Let's start with the simplest example: a non-variable font (no public axes) which uses smart components to speed up design. Hangul is a good example: many syllables are made up of the same components, but those components may have different shapes depending on context.

Let's imagine a Hangul font with two *composite glyphs*: na (나) and nan (난). Both are made up of the same two *components*: the base consonant `n` (ᄂ) and the vowel `a` (ㅏ). `na` uses both components at full height, while `nan` uses three components at half height, and with the `n` at the bottom flatter. So we could design `n` with two *private axes*: `height` (0 to 1) and `flatness` (0 to 1), and `n` with one *private axis*: `height` (0 to 1).

We draw the following masters:

* `n`: Four masters (the 2×2 grid of `height` × `flatness`)
* `a`: Two masters (`height=0` and `weight=1`)

Then we define the composite glyphs:

* `na`:
    * Component `n`: at (0, 0), instantiated at `height=0.8`, `flatness=0`
    * Component `a`: at (500, 0), instantiated at `height=1`
* `nan`:
    * Component `n`: at (0, 250), instantiated at `height=0.5`, `flatness=0`
    * Component `a`: at (500, 250), instantiated at `height=0.5`
    * Component `n`: at (250, 0), instantiated at `height=0.5`, `flatness=1`

How would we represent this in VARC?


## What goes where

The VARC table is designed to work *alongside* the existing `glyf`/`gvar` or `CFF2` tables, not replace them. Here's how the responsibilities are divided:


### In `fvar`:

**All axes** - both public *and* private axes - must be declared in `fvar`. In this case, there are no public axes. The private axes - `height` and `flatness` - are marked as "hidden" (min=default=max, or using the hidden flag). Each unique private axis in a font is assigned a tag; since this isn't user-facing it doesn't matter what it is, so we can use a counter.

In our example, `fvar` would contain:

* axis 0: `0000` (or similar tag, -1 to 1, hidden) - for `height`
* axis 1: `0001` (or similar tag, -1 to 1, hidden) - for `flatness`

Note that although in our design space these private axes run from 0 to 1, in `fvar` they are defined from -1 to 1. It's just what we do.


### In `glyf`/`gvar` (or `CFF2`):

**Component base glyphs with their variations** - the actual outline data for `n` and `a`. These are stored as "flat" glyphs - regular variable outlines in the traditional OpenType way. The `gvar` table stores the deltas for how these outlines change across their design space.

**Empty placeholder for composite glyphs** - `composite` gets an *empty* entry in `glyf` (no contours), because its visual appearance is entirely composed of components defined in VARC.


### In the `VARC` table:

This is where we tell the renderer how to build `na` and `nan` from their components, and at which axis values to instantiate those components.

The VARC table looks like this:


```
struct VARC {
  uint16 majorVersion;     // 1
  uint16 minorVersion;     // 0
  Offset32To<Coverage> coverage;              // Which glyphs have VARC data
  Offset32To<MultiItemVariationStore> varStore;  // Variations for transforms/locations
  Offset32To<ConditionList> conditionList;    // Optional conditional components
  Offset32To<CFF2IndexOf<TupleValues>> axisIndicesList;  // Shared axis index tuples
  Offset32To<CFF2IndexOf<VarCompositeGlyph>> glyphRecords;  // The composite definitions
};
```


The VARC table begins with major and minor version fields which are set to 1.0 and then a coverage table, just like `GSUB` and `GPOS` subtables, which tell us which glyphs have VARC data. The glyph indices go in the coverage table, and this is used to index the `glyphRecords` array. Let's say our font glyph order is:


```
0: .notdef
1: n
2: a
3: na
4: nan
```


Then our `VARC` table coverage table would list glyphs 3 and 4 (`na` and `nan`). We'll skip over the other fields for now and look at the glyph records.

`glyphRecords` is an offset to a `CFF2IndexOf&lt;VarCompositeGlyph>`, where `CFF2IndexOf` just means "cleverly encoded array". So we have an array of `VarCompositeGlyph` records, one per *composite glyph*, indexed by its order in the coverage table: `na` is the first entry in the array, `nan` is the second.

The `VarCompositeGlyph` record is also pretty simple; it's just a list of `VarComponent` records, one per component. The `VarComponent` record is where the magic happens.


### The VarComponent record

The VarComponent record describes how to instantiate a single component glyph within a composite glyph. It includes:



* Flags indicating which fields are present and how to interpret them
* The glyph ID of the component
* An index into the shared `axisIndicesList` indicating which axes this component uses
* The default axis values for this component

Here's what it looks like:


```
struct VarComponent {
  uint32 flags;               // See below
  GlyphID16 gid;              // The component glyph ID. Can also be a GlyphID24!
  uint32 axisIndicesIndex;    // Index into axisIndicesList
  TupleValues axisValues;     // Default axis values for this component
  uint32 axisValuesVarIndex;  // Index into varStore for axis value deltas
  uint32 transformVarIndex;   // Index into varStore for transform deltas
  // A variable number of transformation fields follows dependent on flags
}
```


Here are the list of flags; you don't have to take them in now but I'm listing them here so that you don't get surprised when you see them later:


| bit number | name |
|-|-|
| 0 | `RESET_UNSPECIFIED_AXES` |
| 1 | `HAVE_AXES` |
| 2 | `AXIS_VALUES_HAVE_VARIATION` |
| 3 | `TRANSFORM_HAS_VARIATION` |
| 4 | `HAVE_TRANSLATE_X` |
| 5 | `HAVE_TRANSLATE_Y` |
| 6 | `HAVE_ROTATION` |
| 7 | `HAVE_CONDITION` |
| 8 | `HAVE_SCALE_X` |
| 9 | `HAVE_SCALE_Y` |
| 10 | `HAVE_TCENTER_X` |
| 11 | `HAVE_TCENTER_Y` |
| 12 | `GID_IS_24BIT` |
| 13 | `HAVE_SKEW_X` |
| 14 | `HAVE_SKEW_Y` |
| 15-31 | `RESERVED_MASK`. Set to 0 |


Let's build this up from very simple pieces. If you just want to reference a component glyph with no variation, you would set `flags` to 0, provide the `gid`, and leave the rest empty. This would just render the component glyph as-is.


    *Special feature!* Sometimes we design a glyph as a mix of components and outlines. Before VARC, OpenType didn't support that, so the font compiler decomposes the components into outlines and merges them into the base glyph. But with VARC we can express this by putting the outlines into the composite glyph's record in the `glyf` table and then, in the `VARC` table, adding a `VarComponent` record with the `gid` pointing to the *composite glyph itself* in the `glyf` table as well as `VarComponent` records for any other components. The renderer will iterate through the `VarComponent` records, rendering each one in turn, one of which will be the composite glyph's own outlines.

Now let's add a simple transformation. There are bit flags you set to indicate which transformation fields are present. For example, if you want to translate the component by `(100, 0)`, you would set the `HAVE_TRANSLATE_X` flag and provide a `translateX` field with value `100`. Job done.


### Instantiating components

And now, let's instantiate our component glyphs at their private axis values. Reminding ourselves of what we're trying to achieve:


* `na`:
    * Component `n`: at (0, 0), instantiated at `height=0.8`, `flatness=0`
    * Component `a`: at (500, 0), instantiated at `height=1`

So we will have two `VarComponent` records in the `VarCompositeGlyph` for `na`. We'll start with `n`.


#### VarComponent flags

The first step in writing the VarComponent record for `n` is to determine the flags by asking ourselves some questions. I'm not going to ask the question about `RESET_UNSPECIFIED_AXES` yet because it's complicated but I'm going to tell you that the answer is "no".

Do we `HAVE_AXES`? That is, does this component glyph have some variation axes that we need to care about in this instantiation? Yes, it has two private axes: `height` and `flatness`. So we set bit 1.

Are we going to vary the axis values across different masters of the composite glyph? No, this isn't a variable font, there's no variation, so `AXIS_VALUES_HAVE_VARIATION` is not set. Similarly we're not varying the transform across the font's designspace because the font doesn't actually have a designspace, so `TRANSFORM_HAS_VARIATION` is not set.

What transformation elements do we have? For `n` we're just placing it at (0, 0) so we've got nothing. Leave those flags clear.

Is the gid 24 bit? No, because this is a tiny dummy font and we're using `GlyphID16`. So our flags are just `HAVE_AXES` = `0b00000010` = 2.


#### VarComponent gid

It's just the gid of `n`, which is 1. Easy.


#### Storing the axis values

Font-wide we have a total of two axes, both of which are private. If you imagine a full-sized font with many glyphs, and quite a few private axes per glyph, *not all components will use all font axes*; they'll just use the ones that are relevant to instantiating those components. In our case we have:



* Composite glyph `na` component `n`: uses `height` and `flatness`
* Composite glyph `na` component `a`: uses `height`
* Composite glyph `nan` component `n`: uses `height` and `flatness`
* Composite glyph `nan` component `a`: uses `height`
* Composite glyph `nan` component `n`: uses `height` and `flatness`

You can see there are two distinct combinations of axes used: `[height, flatness]` and `[height]`. In a bigger font you might have `[height, flatness, serifStyle]` or `[flatness, curl]`, `[height, serifStyle, curl]`, etc. It would be wasteful to store all the axis values for *all* the font axes for each component, when most components only care about a few axes. So we organize the distinct combinations of axes into a list called the `axisIndicesList`. Using the index of each axis in the `fvar` table's list of axes, we build up the `axisIndicesList` like so:


```
axisIndicesList[0] = [0, 1]     // height, flatness
axisIndicesList[1] = [0]        // height
```


We're currently creating the `n` VarComponent for `na`, which uses both `height` and `flatness`, so its `axisIndicesIndex` is `0`.

Our *axis values* for `n` in `na` are:



* `height=0.8`
* `flatness=0`

These get stored in the `axisValues` field as a `TupleValues` structure, which is just an array of normalized F2DOT14 values. As an endless opportunity for fiddly bugs, we need to remember that our private axes are defined in `fvar` to run from -1 to 1 but that our design space for these private axes is from 0 to 1, so we have to normalize these values like so:



* `height=0.8` → normalized = `(0.8 - 0) / (1 - 0) * 2 - 1 = 0.6`
* `flatness=0` → normalized = `(0 - 0) / (1 - 0) * 2 - 1 = -1.0`

We aren't varying the axis values today, so there’s no `axisValuesVarIndex` field.

Putting it all together, the VarComponent record for `n` in `na` looks like this:


```
VarComponent {
  uint32 flags = 2;                     // HAVE_AXES
  GlyphID gid = 1;                      // n
  uint32 axisIndicesIndex = 0;          // height, flatness
  TupleValues axisValues = [0.6, -1.0]; // height=0.8, flatness=0
  // no transform fields
}
```


Similarly we can now do `a` in `na`. It uses a different axis combination, so its `axisIndicesIndex` is `1`. Its axis value for `height=1` normalizes to `1.0`. And we have a translation transformation. So its VarComponent record looks like this:


```
VarComponent {
  uint32 flags = 6;                    // HAVE_AXES | HAVE_TRANSLATE_X
  GlyphID gid = 2;                     // a
  uint32 axisIndicesIndex = 1;         // height
  TupleValues axisValues = [1.0];      // height=1
  FWORD translateX = 500;              // translateX=500
}
```


That's the `na` composite glyph done! The `nan` glyph is similar; you just have to remember to set the correct axis values for each component instance. We will have three VarComponent records for `nan`, like this:


```
// n, `height=0.5`, `flatness=0`, at (0, 250)
VarComponent {
  uint32 flags = 34;                    // HAVE_AXES | HAVE_TRANSLATE_Y
  GlyphID gid = 1;                      // n
  uint32 axisIndicesIndex = 0;          // height, flatness
  TupleValues axisValues = [0.0, -1.0]; // height=0.5, flatness=0
  FWORD translateY = 250;               // translateY=250
}

// a, `height=0.5`, at (500, 250)
VarComponent {
  uint32 flags = 50;                    // HAVE_AXES | HAVE_TRANSLATE_X | HAVE_TRANSLATE_Y
  GlyphID gid = 2;                      // a
  uint32 axisIndicesIndex = 1;          // just height 
  TupleValues axisValues = [0.0];       // height=0.5
  FWORD translateX = 500;               // translateX=500
  FWORD translateY = 250;               // translateY=250
}

// n, `height=0.5`, `flatness=1`, at (250, 0)
VarComponent {
  uint32 flags = 6;                     // HAVE_AXES | HAVE_TRANSLATE_X
  GlyphID gid = 1;                      // n
  uint32 axisIndicesIndex = 0;          // height, flatness
  TupleValues axisValues = [0.0, 1.0];  // height=0.5, flatness=1
  FWORD translateX = 250;               // translateX=250
}
```


And that's it. Simple, right?


## I heard you liked variable glyphs so I put variable glyphs in your variable glyphs

Hah, fooled you, because now let's add user-facing variation axes into the mix.

Let's extend our example font to have a public `wght` axis (400-700) which affects the width and positioning of both `n` and `a`. Additionally, let's say that the `a` component varies in (public) `wght` as well as private `height`. (In real life, the `n` component would probably also vary in `wght`, but let's keep it simple.) We'll also say that in the `na` (I'm not going to do `nan` in this example) the `n` gets slightly *flatter* as it gets heavier.



* At weight=400, `n` is positioned normally, at `height=1`, `flatness=0`
* At weight=700, `n` is positioned at x=-50, at `height=1`, `flatness=0.4`

Off we go again. Let's build `fvar` first:



* axis 0: `wght` (400-700, visible)
* axis 1: `0000` (or similar tag, -1 to 1, hidden) - for `height`
* axis 2: `0001` (or similar tag, -1 to 1, hidden) - for `flatness`

Now we build the `glyf` table as well as a `gvar` table with variation data for `n` and `a` across `wght`, `height`, and `flatness`. `na` and `nan` get no outlines as before. Nothing new here.

Now we're ready to build the VARC table. Coverage is the same as before. Axis indices are a bit different, but only because the indices of our axes have changed. We're not varying any component axis values by weight, so we only have the same two axis combinations as before:


```
axisIndicesList[0] = [1, 2]     // height, flatness
axisIndicesList[1] = [1]        // height
```


Time to create our `VarComponent` records for `na`. We'll start with `n`, and as before we start by asking the flags questions.

Now we *can* ask about `RESET_UNSPECIFIED_AXES`. This question asks: "`n` varies in `wght`, but we're not passing a `wght` value to it specifically. What should the `wght` value be for instantiation? Should it inherit the current `wght` value from the font's settings, or should it reset to the default `wght` value (400)?" In this case we want to "pass through" the current `wght` value into the component so that it can vary normally according to the user-facing settings, so we leave this flag clear. If we wanted to force the component to always use the default `wght` value, we would set this flag.

Next question, do we `HAVE_AXES`? Yes, we are setting `height` and `flatness`, so bit 1 is set. Are we varying the *axis values* across masters? Yes, because `flatness` changes from 0 to 0.4 as `wght` goes from 400 to 700, so bit 2 is set.

Now the transform. Are we varying the *transform* across masters? Yes, because `n` moves from x=0 to x=-50 as `wght` goes from 400 to 700, so bit 3 is set. And the only transform we're using is `translateX` (even though it is zero at the default location, we're going to vary it so we need to specify it), so bit 4 is set. We set the flags to `HAVE_AXES | AXIS_VALUES_HAVE_VARIATION | TRANSFORM_HAS_VARIATION | HAVE_TRANSLATE_X` = `0b00011110` = 30.

On to the axis values. First we store the axis values at their default locations:



* `height=1` → normalized = `(1 - 0) / (1 - 0) * 2 - 1 = 1.0`
* `flatness=0` → normalized = `(0 - 0) / (1 - 0) * 2 - 1 = -1.0`

And now we store the deltas for the axis values, explaining how the axis values change across `wght`:



* `height`: stays at 1.0 across all weights, so deltas are [0, 0, 0]
* `flatness`: goes from 0 to 0.4 as `wght` goes from 400 to 700, so deltas are [0, 0.2, 0.4]

We hand these tuples to our MultiItemVariationStore builder and get back an index which we store in `axisValuesVarIndex`.

Now the transform. The default `translateX` is 0, so we store that in the component record. The deltas for `translateX` as `wght` goes from 400 to 700 are [0, 0, -50]. We hand these to our MultiItemVariationStore builder and get back an index which we store in `transformVarIndex`.

Putting it all together, the VarComponent record for `n` in `na` now looks like this:


```
VarComponent {
  uint32 flags = 30;                   // HAVE_AXES | AXIS_VALUES_HAVE_VARIATION | TRANSFORM_HAS_VARIATION | HAVE_TRANSLATE_X
  GlyphID gid = 1;                     // n
  uint32 axisIndicesIndex = 0;         // height, flatness
  TupleValues axisValues = [1.0, -1.0]; // height=1, flatness=0
  uint32 axisValuesVarIndex = X;       // index into varStore for axis value deltas
  uint32 transformVarIndex = Y;        // index into varStore for transform deltas
  FWORD translateX = 0;                // translateX=0
}
```


See, that wasn't too bad, was it?


# Additional wrinkles


## RESET_UNSPECIFIED_AXES

The `RESET_UNSPECIFIED_AXES` flag is useful when you have a component glyph that varies in some public axes, but you want to force it to always use the default value for those axes when instantiated within a composite glyph. For example, imagine a component glyph that varies in `wght` and `opsz`, but you want it to always use the default `opsz` value when used in a composite glyph, you would set the `RESET_UNSPECIFIED_AXES` flag. This tells the renderer to ignore the current `opsz` value and use the default instead.


## Conditional components

The VARC table supports conditional components, which are components that are only included in the composite glyph under certain conditions. This is useful for things like ligatures or contextual alternates. The `ConditionList` structure in the VARC table defines the conditions under which each conditional component is included. Each `VarComponent` record can reference a condition from the `ConditionList` using the `HAVE_CONDITION` flag and a condition index.


## Transforms

The VARC table supports a variety of transformations that can be applied to component glyphs, including translation, scaling, rotation, and skewing. Each transformation type has its own flag in the `VarComponent` record, and the corresponding transformation values are stored in the record. Additionally, transformations can be varied across the font's design space using the `TRANSFORM_HAS_VARIATION` flag and a variation index.


## MultiItemVariationStore

Don't. Let someone else deal with it.
