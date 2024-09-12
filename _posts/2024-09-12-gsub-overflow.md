---
published: true
layout: post
title: "Avoid GSUB Overflows"
---

Here's some techniques adapted from an email I sent recently - a project with complicated contextual substitution rules was having problems with GSUB tables overflowing and refusing to compile. Follow me for a deep dive into contextual chaining rules, binary layout, and why compilers are stupid so we have to be clever...

This is a problem we came across a lot in the making of the Gulzar Nastaliq font. The problem comes with large chained contextual lookups.

OpenType has three ways of storing these lookups in the binary, Format 1, Format 2 and Format 3. Format 1 is used for simple contexts where there are no classes. Format 2 is used for non-overlapping classes. Format 3 is the most flexible - it is used for anything else, i.e. sets of rules where the classes overlap.

But format 3 is very inefficient; it stores a list of glyphs for every position for every rule. For large contextual lookups, you really need them to be stored as format 2.

So how do we do this? Split the rules up into three parts, lookbehind, input sequence and lookahead. For each of these three parts (separately), each position in the sequence must be a non-overlapping class and all glyphs must be in the same class. For example, this is OK:

```fea
        # Lookbehind    # Input   # Lookahead
 	sub [A B] [C D]     [x y]'    E;
 	sub [A B] [E F]     z'        F;
```

In the lookbehind, there are three different classes at each position: `[A B]`, `[C D]` and `[E F]`. In the input, there are two different classes: `[x y]` and `[z]`.

> Important note: when something is stored as format 2, everything is a class, even single glyphs. So by "class" we just mean "the set of glyphs at a given position in a rule.

In the lookahead, there are two different classes `[E]` and `[F]`. `E` and `F` are in the same class in the look _behind_, but remember we are considering each part - lookbehind, input, lookahead - separately. They have separate class definitions stored in the lookup.

So having everything separated out like that, the compiler can assign each position to a class like this:

```fea
sub @lookbehind_1 @lookbehind2 @input_1' @lookahead_1;
sub @lookbehind_1 @lookbehind3 @input_2' @lookahead_2;
```

and _each glyph is only in one class_ per part. All the conditions are met and this will be stored as a format 2 lookup, and all will be well.

But if we do this:

```fea
        # Lookbehind    # Input   # Lookahead
 	sub [A B] [C D]     [x y]'    E;   # Rule 1
  	sub [A B] [E D]     z'        F;   # Rule 2
```

Then we have a problem - we now have three different lookahead classes (`[A B]`, `[C D]`, `[E D]`) but glyph `D` appears in both of them. Overlapping classes cannot compile to format 2, so we lose.

To fix this we can split things up:

```fea
        # Lookbehind    # Input   # Lookahead
	sub [A B] C         [x y]'    E;   # Rule 1a
 	sub [A B] D         [x y]'    E;   # Rule 1b
 	sub [A B] E         z'        F;   # Rule 2a
 	sub [A B] D         z'        F;   # Rule 2b
```

It means the same, we now have four lookbehind classes (`[A B]`, `[C]`, `[D]`, `[E]`) but each glyph only appears in one class. Nothing overlaps so we can compile to format 2.

Another example:

```fea
 	@Class1 = [ A B ];
 	@Class2 = [ C D ];
 	@Class3 = [ E F ];

        # Lookbehind       # Input  # Lookahead
 	sub @Class1'           @Class1' @Class2; # Rule 1
 	sub @Class2'           @Class2' @Class1; # Rule 2
 	sub [@Class2 @Class3]' @Class3' @Class3; # Rule 3
```

This is OK, right? Each glyph is in a separate class, nothing overlaps, so this can be format 3? No! AFDKO "classes" are just for _our_ use. The _compiler_'s classes are what matters.

What the compiler sees is that the lookbehind sequence to rule 2 starts with `[C D]` and the lookbehind sequence to rule 3 starts with `[C D E F],` and `C` and `D` are in both so they are overlapping, so can't be expressed as format 2. We must split it up again:

```fea
 	sub @Class1'           @Class1' @Class2; # Rule 1
 	sub @Class2'           @Class2' @Class1; # Rule 2
 	sub @Class2'           @Class3' @Class3; # Rule 3a
 	sub @Class3'           @Class3' @Class3; # Rule 3b
```

Now we are OK.

> This is terrible! Why can't the compiler do this for us? Well, in theory, it can, yes. But in practice working out how to do the splits in the general case is horribly complicated, and more to the point, _AFDKO feature syntax is binary equivalent_. It's a deliberately low-level language to allow for simple compilation. It was never really meant for this kind of stuff.
> (Sidenote: For the Gulzar Nastaliq project, I wrote Python code which wrote feature code and which made sure that no classes overlapped. Which is a good way to deal with things if you're controlling the process; nobody should be writing complicated feature code by hand, really. But we don't have a general-purpose optimizing feature compiler which takes hand-written code and makes it better. We have to use our brains instead.)

Let's take a practical example. A large project had a "glyph shuffler" which first replaces some glyphs with alternates, but having done so, then tries to avoid getting the same set of alternates within the same word. So there were rules like

```fea
sub @dflt @All_Letters @dflt' by @alt1;
sub @alt1 @All_Letters @alt1' by @alt2;
sub @alt2 @All_Letters @alt2' by @alt3;
sub @alt3 @All_Letters @alt3' by @dflt;

sub @dflt @All_Letters @All_Letters @dflt' by @alt1;
sub @alt1 @All_Letters @All_Letters @alt1' by @alt2;
sub @alt2 @All_Letters @All_Letters @alt2' by @alt3;
sub @alt3 @All_Letters @All_Letters @alt3' by @dflt;

# ... and so on up to seven letters
```

A huge lookup, which is going to overflow. It can't be expressed as format 2 because of the `@All_Letters` class: the glyphs in `@dflt` and `@alt1` are also going to be in `@All_Letters`, so you will have overlapping classes, so you will get a format 3 subtable.

The usual answer would be to rewrite the rules to split `@All_Letters` into non-overlapping classes:

```
sub @dflt @dflt @dflt' by @alt1; # Rule 1a
sub @dflt @alt1 @dflt' by @alt1; # Rule 1b
sub @dflt @alt2 @dflt' by @alt1; # Rule 1c
sub @dflt @alt3 @dflt' by @alt1; # Rule 1d
```

But we cannot do that here. Any instance of `@All_Letters` explodes into 4 combinations, and a rule like:

```fea
sub @alt3 @All_Letters @All_Letters @All_Letters @All_Letters @All_Letters @alt3' by @dflt;
```

would need to be split into 4x4x4x4x4=4096 rules. (That's bad enough, and the real code in the project was much worse.)

How would I approach this problem? The first thing to do is to create some utility lookups:

```fea
lookup dflt_to_alt1 { sub @dflt by @alt1; } dflt_to_alt1;
lookup alt1_to_alt2 { sub @alt1 by @alt2; } alt1_to_alt2;
```

> Sidenote: You should _always_ start like this. Adobe's "inline contextual substitutions" are a pain in the neck. For any kind of complicated OpenType coding, you should always prefer creating an auxiliary lookup and using `sub glyph' lookup LookupName` in your main rule instead. It literally compiles to the same code, but you get more control over how the lookups are laid out, you avoid creating duplicate lookups, and it's _much_ easier to reason about.

Next, some more utility lookups. They start at a glyph, move two places ahead, and then try to apply one of the lookups we just defined. And here's the trick: these ones apply to _all_ letters.

```fea
lookup third_letter_to_alt1 {
  sub @All_Letters' @All_Letters' @All_Letters' lookup dflt_to_alt1;
} third_letter_to_alt1;

lookup third_letter_to_alt2 {
  sub @All_Letters' @All_Letters' @All_Letters' lookup alt1_to_alt2;
} third_letter_to_alt2;
```

And finally we can write the first two rules of our main routine like this:

```
sub @dflt' lookup third_letter_to_alt1;
sub @alt1' lookup third_letter_to_alt2;
```

Think about what happens when this is processed. We look for the first glyph in our sequence; if it's a default letter, then we go to `third_letter_to_alt2`, which matches three _general_ sets of glyphs, and then on the third class it jumps to another routine, `dflt_to_alt1`. This lookup will only substitute glyphs in `@dflt`, so the fact that we jumped there after matching `@All_Letters` doesn't matter.

Having written our lookups this way, there are no overlapping classes. All of these contextual lookups will be stored as format 3 and so will be much smaller.

I wish we didn't have to write our feature code with the binary layout in mind, and it would be great if the compiler could do this for us, but it can't, so we have to do a lot of thinking and planning to make sure that each lookup has non-overlapping classes. In general, I find that chaining lookups as we have done above - start in one place, then jump to another lookup to match something else, then jump from there to another lookup - is a good technique to help us get the format that we want.

Bonus tips:

- When you look at the TTX dump of the GSUB table, look for lines like `<ChainContextSubst index="<something>" Format="3">` - that will tell your where the problems are likely to be.

- If you turn on the environment variable `FONTTOOLS_LOOKUP_DEBUGGING=1` in your shell before using `fontmake`, the TTX dump will be annotated with the locations in your feature file where the lookups are declared, making it easier to trace back into your source code and find the problem. (Also give all your sets of lookups names; you do that already, right?)

- I've released a [Python script](https://github.com/simoncozens/font-engineering/blob/master/lookup-size.py) in my font engineering tools repo which can report on large lookups, which will help you to know where to start optimizing.
