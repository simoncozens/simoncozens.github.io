---
published: true
layout: post
title: Better Fonts Through Test-Driven Development
---

Fonts are, increasingly, pretty complex pieces of software. I work primarily on the layout side, creating (both manually and semi-automatically, through scripting) large collections of OpenType shaping rules for fonts with complex layout requirements. But writing the rules is half the battle. How do you know they work? How do you ensure that, at the end of the day, the collection of rules you've written actually produces the results that you expect, in all the cases that you expect?

More to the point, when you have a large collection of (possibly interacting) rules, how do you know that they play nicely together? How do you ensure that you can add new rules without breaking all the code you've written so far, that fixing a problem with _this_ glyph sequence doesn't then accidentally affect the outcome of _that_ glyph sequence? Sometimes with so many disparate but interlocking rules, it can feel like adding one more brick to the top might cause the whole Jenga tower to come crashing down around you.

It all comes down to _confidence_. How can you have confidence in your layout rules?

Thankfully, this is a (largely) solved problem in computer science, and the answer is something called _TDD_.

### What is TDD and why do I want it?

The idea behind TDD, test-driven development, is pretty simple:

- _First_ you write a test for the behaviour you want to see.
- _Then_ you implement the behaviour in code.
- _Finally_ run your test suite to make sure that your implementation did what it should.

The order is important here. You first write a test which fails, then you make it pass. You build the rules _around_ the behaviour you want, and _every_ rule has a test. If you do this process for every piece of behaviour you want to see, then you end up constructing a rigorous test suite, full of all the possible situations that you care about. This means that you can determine, scientifically, in an automated way, whether or not your code is behaving properly.

I don't think this is common for font projects. Typically, we make proofs containing the sample texts we're interested in, and we look at them by eye, and see if there are any problems. But this is error-prone. Yes, in one sense, the eye is the ultimate arbiter of the font. But the eye gets tired, or falls into patterns. How do we know we covered all the rules? How do we know we covered all the possible combinations of input text? How do we know that we haven't got too familiar with what we're looking at that we haven't missed a glaring mistake? A more mechanical approach can find problems that the eye might miss.

### What does it look like?

For my font development, I use an automated test harness for OpenType shaping that can mechanically check for any of three desirable properties:

- That a given string, when run through a shaping engine, produces a particular, predetermined result. I can test that ကြ produces `medialRa-myanmar.w2=0+216|ka-myanmar=0+1040` but that ကြု produces `medialRa_uMark-myanmar.w2=0+216|ka-myanmar=0+1040`.
- That a given string _or pattern of strings_ does not produce any glyphs in a given set. For example, `Consonant U+1039 Consonant` should produce a stacked sequence, but if the shaping output ever includes the visible virama `virama-myanmar`, then the stacking has failed.
- That a given string _or pattern of strings_ does not create any glyphs which collide with other glyphs. In a Nastaliq font, `ThingsWithDotsBelow Kasra? ThingsWithDotsBelow Kasra? TrickyFinalCharacters` should not create any clashes.

The current system in fontbakery is builds on two previous efforts, my own [gnipahs](https://github.com/simoncozens/font-engineering/blob/master/gnipahs.py) which using Harfbuzz and collidoscope to check shaping expectations and detect collisions, and Nikolaus Waxweiler's work (in a private project) which added robust JSON syntax to the shaping files and more functionality such as the ability to select OpenType features.

These three tests are run within Google Fonts' [fontbakery](https://github.com/googlefonts/fontbakery) font QA tool, and any test failures are reported as part of fontbakery's HTML report.

The report looks like this, not just giving me an automated result (these ten tests failed!) but also giving me visual feedback of what went wrong, as well as what it _should_ have looked like:

![noto-regression.png](/images/noto-regression.png)

Obviously, this font is not done yet. But when the report tells me that all those tests are passing, I can have _confidence_ that my font behaves the way it is supposed to.

### How do I get it?

OK, so you're convinced and you want in. What do you need to do?

- First, you need to get a copy of fontbakery, which contains the shaping test suite runner! This is not _quite_ released yet, but you can get it from git:

```
pip3 install -U git+https://github.com/googlefonts/fontbakery@4d680af
```

- Next, you will need to tell fontbakery where your tests will live. Create a fontbakery configuration file in either TOML or YAML format, (I'm going to use YAML here.) which looks like this:

```YAML
com.google.fonts/check/shaping:
    test_directory: qa/shaping_tests
```

- Now you can write your test files. These will be placed in the `qa/shaping_tests/` subdirectory of your font project. Each of these test files will be a JSON file. Within a JSON file, you can mix and match each of the three test-types we mentioned above. The JSON file will have the following basic structure:

```JSON
{
  "configuration": {
    "collidoscope": { /* Configuration for collision tests, if any */ },
    "forbidden_glyphs": [ /* list of forbidden glyphs, if any */ ],
    "defaults": { /* Properties which apply to all tests */ },
    "ingredients": { /* Definitions of patterns to be used in the tests, if any */ }
  },
  "tests": [
    {
      /* A test goes here */
    },
    {
      /* A test goes here */
    }
  ]
}
```

Within a given test, the relevant keys are:

- `input`: Text to be shaped. If `collidoscope` is configured, then a collision test is run on this text; if `forbidden_glyphs` is configured, then any glyphs in that list must _not_ appear in the shaped output.
- `expectation`: A `hb-shape`-like string showing what the shaping output ought to look like. This may contain positioning information taken from `hb-shape` (e.g. `ka=0+1124|ta.sub=0@-235,0+0`) if you want a "full" test, or you can just give the glyph names if you are interested in making sure that the substitution rules are working and you don't care about positioning at this stage (e.g. `ka|ta.sub`). If you _do_ give a full positioning test, you get visual feedback on what the expectation should look like.
- `features`: Any OpenType features to be applied to this test.
- `input_type`: If this is set to "pattern", then a pattern-based test will be run.

So, for example, the following JSON file:

```
{
  "configuration": {
    "collidoscope": { "area": 0, "marks": true },
    "forbidden_glyphs": [".notdef", "virama-myanmar", "uni25CC"],
    "defaults": {
      "allowedcollisions": [
        "medialYa-myanmar/aaSign-myanmar",
        "medialWa-myanmar/medialYa-myanmar.bt1"
      ]
    }
  },
    {
      "input": "ကါံ",
      "expectation": "ka-myanmar|anusvara-myanmar|tallAa-myanmar"
    },
}
```

will do the following:

- Shape the text `ကါံ` and ensure that the output buffer is `ka-myanmar|anusvara-myanmar|tallAa-myanmar`.
- Check that when the test is shaped, the glyphs `.notdef`, `virama-myanmar` and `uni25CC` do not appear in the output.
- Ensure that when the test is shaped, no glyphs interfere with one another _except_ the glyph sequences `medialYa-myanmar/aaSign-myanmar` and `medialWa-myanmar/medialYa-myanmar.bt1` (which are allowed to form overlaps).

`expectation` is optional. If you just want to apply the collision/forbidden glyph tests, then don't provide one.

_Pattern-based tests_ are a way to shape a wide range of related strings and run collision and forbidden glyph tests on them, without having to spell out every single combination. For example, the following JSON file:

```
{
  "configuration": {
    "collidoscope": { "area": 0, "marks": true, "faraway": true },
    "forbidden_glyphs": [".notdef", "virama-myanmar", "uni25CC"],
    "defaults": {
      "allowedcollisions": [
        "medialYa-myanmar/aaSign-myanmar",
        "medialWa-myanmar/medialYa-myanmar.bt1",
        "medialRa-myanmar.tt1/medialYa-myanmar",
        "medialRa-myanmar.w2.tt1/medialYa-myanmar",
        "medialRa-myanmar/medialYa-myanmar",
        "medialRa-myanmar.w2/medialYa-myanmar"
      ]
    },
    "ingredients": {
      "Consonant": "[ကဟဂ]",
      "MedialRa": "ြ",
      "Asat": "်",
      "MedialYa": "ျ",
      "MedialWa": "ွ",
      "MedialHa": "ှ",
      "VowelBottom": "[ုူ]"
    }
  },
  "tests": [
    {
      "input_type": "pattern",
      "input": "Consonant Asat? MedialYa? MedialRa? MedialWa? VowelBottom?"
    }
  ]
}
```

will shape `3 * 2 * 2 * 2 * 2 * 3 = 144` individual strings, comprising of each of the three consonants in the list, with and without an asat, with and without a medial ya, with and without a medial ra, with and without a medial wa, and with no below vowel, a u vowel and a uu vowel.

For each of these 144 strings, we check whether any forbidden glyphs were produced, and whether any of the glyphs interfered with one another in unexpected ways. With a sufficiently well-designed set of patterns you can test every possible combination of inputs - one of my fonts runs thousands of these tests from a test file of a few dozen lines.

After a while, you will have a directory full of JSON files with your tests in. (I find it useful to have separate JSON files to test different things; one to test regressions for rules I am creating, one to test sequences for forbidden glyphs, one to test sequences for collisions. It's even useful to have separate files for particular families of rules - for example, I have one called `medial-ra-and-friends.json` which exhaustively tests that medial ra shapes in a variety of different circumstances.)

You have your test files. You have your `fontbakery.config` YAML file. Now to run the test!

```
fontbakery check-profile --config fontbakery.config --html report.html fontbakery.profiles.shaping My-Font.ttf
```

You will get a _lot_ of output on the console, but hopefully you will also get a pretty report in `report.html` similar to the one above.

Running these tests gives me _confidence_ that the font does what I expect it to do. Having a robust test suite gives me confidence that, each time I add a new rule, not only does the new rule do what I expect, it also does not do so in a way that interferes with behaviour that was working previously.

For anyone working with complex layout rules, TDD is going to be an absolute gamechanger.

Shape your text with _confidence_ using test-driven development!

#### Related projects

There are also some attempts to apply the _Test Driven Development_ philosophy to other aspects of typeface design.

- https://github.com/SorkinType/EQX
- https://github.com/typefacedesign/document-driven-typedesign
- Dalton Maag's "Scope One": [github.com/daltonmaag/scope-one](https://github.com/daltonmaag/scope-one) (Please note their JSON test suites are a different format to the ones described above, so don't try using them with fontbakery!)
