---
published: true
title: A Duffer's guide to Fontconfig and Harfbuzz
layout: post
---

I'm working on switching the font shaping part of <a href="http://www.sile-typesetter.org/">SILE</a> to use Harfbuzz instead of Pango because <a href="https://github.com/simoncozens/sile/issues/8">reasons</a>, and have found myself a bit hampered by the <a href="http://stackoverflow.com/questions/21317181/harfbuzz-getting-started">lack of useful documentation</a>. To be fair, if you actually build HB from source you get an auto-generated API reference, but there's nothing really explaining how to go from a string of characters to a set of glyph positioning information, which is a shame because that is what Harfbuzz is for.

The first obstacle I hit when moving off Pango is that Pango allows you to talk about font names, whereas when dealing with Harfbuzz directly you have to locate and open the font file yourself; the usual way to do this is with Fontconfig. The Fontconfig <a href="http://www.freedesktop.org/software/fontconfig/fontconfig-devel/t1.html">developer documentation</a> similarly provides a reference to the API functions, but nothing about going from a font description to a file name, which is a shame because that is what Fontconfig is for.

So I had to go around and gather various bits of information from mailing list posts and StackExchange questions and... here is one solution to the problem of turning text into positioning information, and turning font descriptions into filenames. I don't claim it's the best, but it works.

First let's work on the Fontconfig side. Fontconfig works by means of <em>patterns</em>; you specify the features that you want to find in your font database, and it goes and finds fonts which match. The most obvious features you'll want are the font's family name, but you might want to search with reference to other things as well. In order to help us, I'm going to declare a structure which encodes all the font-related options we want to specify, and we'll use the members of this structure to drive Fontconfig and Harfbuzz:

<pre>
typedef struct {
  char* family;
  char* lang;
  double pointSize;
  int weight;
  int direction;
  int slant;
  char* style;
  char* script;
} fontOptions;
</pre>

We'll go through the members in turn later on but for now, here's an example of a font described in that structure:

<source>
    fontOptions f = {
      .pointSize = 12,
      .lang = "en",
      .family = "Gentium Book Basic",
      .script = "latin",
      .direction = HB_DIRECTION_LTR,
      .weight = 200,
    };
</source>

Now let's start writing a function to turn a font description, from the above structure, into a font pathname.

<pre>
#include <fontconfig/fontconfig.h>
static char* get_font_path(fontOptions f) {
  FcResult result;
  FcChar8* filename;
  char* filename2;
  int id;
  FcPattern* matched;
</pre>

The first thing we do is to create a new Fontconfig pattern which is going to store our match information. We load it up with the font family name and point size. Fontconfig has a family of typed functions for adding clauses to a match. We have to convert our strings to special Fontconfig strings, but otherwise this is straightforward:

<pre>
  FcPattern* p = FcPatternCreate();

  FcPatternAddString (p, FC_FAMILY, (FcChar8*)(f.family));
  FcPatternAddDouble (p, FC_SIZE, f.pointSize);
</pre>

Now we will add the slant (roman/italic/etc.) and weight requirements to the pattern:

<pre>
  if (f.slant)
    FcPatternAddInteger(p, FC_SLANT, f.slant);
  if (f.weight)
    FcPatternAddInteger(p, FC_WEIGHT, f.weight);
</pre>

Possible values of <code>FC_SLANT</code> are <code>FC_SLANT_ROMAN</code>, <code>FC_SLANT_ITALIC</code> and <code>FC_SLANT_OBLIQUE</code>. Possible values of <code>FC_WEIGHT</code> will do your head in. Here is a conversion table between CSS and Fontconfig font weight constants:

<table class="table">
<tr><th></th><th>CSS</th><th>Fontconfig</th></tr>
<tr><th>Thin</th><td>100</td><td> FC_WEIGHT_THIN  (0)</td></tr>
<tr><th>Ultralight</th><td>200</td><td> FC_WEIGHT_ULTRALIGHT (40)</td></tr>
<tr><th>Light</th><td>300</td><td> FC_WEIGHT_LIGHT (50)</td></tr>
<tr><th>Normal</th><td>400</td><td> FC_WEIGHT_NORMAL (80)</td></tr>
<tr><th>Medium</th><td>500</td><td> FC_WEIGHT_MEDIUM (100)</td></tr>
<tr><th>Demibold</th><td>600</td><td> FC_WEIGHT_DEMIBOLD (180)</td></tr>
<tr><th>Bold</th><td>700</td><td> FC_WEIGHT_BOLD (200)</td></tr>
<tr><th>Ultra bold</th><td>800</td><td> FC_WEIGHT_ULTRABOLD (205)</td></tr>
<tr><th>Heavy</th><td>900</td><td> FC_WEIGHT_HEAVY(105)</td></tr>
</table>

So just divide by five and... no, wait.

Anyway, now we have a pattern which matches the font that we want: its name, weight, point size, and slant. Next, what we will do is ask Fontconfig to fall back to some default fonts if it doesn't find the one that we're after. We do this by adding more patterns. Fontconfig finds the first match, so if we don't match "Gentium Book Basic", it will find:

<pre>
  FcPatternAddString (p, FC_FAMILY,(FcChar8*) "Times-Roman");
  FcPatternAddString (p, FC_FAMILY,(FcChar8*) "Times");
  FcPatternAddString (p, FC_FAMILY,(FcChar8*) "Helvetica");
</pre>

For my purposes this is enough to ensure a match; for yours it might not be. Now we have a pattern, let's match against our font database:

<pre>
  matched = FcFontMatch (0, p, &result);
</pre>

<code>matched</code> is also an FcPattern, but will be filled with information about the matched font. We can get the information out with similar <code>FcPatternGet...</code> functions:

<pre>
  if (FcPatternGetString (matched, FC_FILE, 0, &filename) != FcResultMatch)
    return NULL;
</pre>

We could have set the <code>FC_FILE</code> property in our pattern, but that would be dumb because that's what we're trying to find out. Instead, we get it, into the <code>&filename</code> pointer. This pointer is allocated by Fontconfig and lasts for the lifetime of the pattern, so we're going to make a copy of it, and then release the patterns we allocated:

<pre>
  filename2 = malloc(strlen(filename));
  strcpy(filename2, (char*)filename);
  FcPatternDestroy (matched);
  FcPatternDestroy (p);
  return filename2;
}
</pre>

So at this point we can go from our font description structure to a filename. Hooray! Except---Harfbuzz expects that fonts come from Freetype, so you need to get Freetype up and running. We'll then turn the font description into a filename, turn that into a Freetype font structure, then turn <em>that</em> into a Harfbuzz font structure:

<pre>
#include <ft2build.h>
#include FT_FREETYPE_H
#include FT_GLYPH_H
#include FT_OUTLINE_H

#include <hb.h>
#include <hb-ft.h>

    int device_hdpi = 72;
    int device_vdpi = 72;
    FT_Library ft_library;
    FT_Face ft_face;
    hb_font_t *hb_ft_font;

    assert(!FT_Init_FreeType(&ft_library));
    font_path = get_font_path(f);
    printf("Found font: %s\n", font_path);
    assert(!FT_New_Face(ft_library, font_path, 0, &ft_face));
    assert(!FT_Set_Char_Size(ft_face, 0, f.pointSize * 64, device_hdpi, device_vdpi ));

    hb_ft_font = hb_ft_font_create(ft_face, NULL);
</pre>

Freetype, bless its heart, uses 1/64th of a font as its fundamental unit of type size. You also need to tell it what DPI your output device is going to be at. I'm using printer's points, so I configure for 72dpi square pixels.

Next up, we create a buffer for Harfbuzz to do its string work in, and set that up the various properties we know about the text:

<pre>
    buf = hb_buffer_create();
    if (f.script)
      hb_buffer_set_script(buf, hb_tag_from_string(f.script, strlen(f.script)));
    if (f.direction)
      hb_buffer_set_direction(buf, f.direction);
    if (f.lang)
      hb_buffer_set_language(buf, hb_language_from_string(f.lang,strlen(f.lang)));
</pre>

Harfbuzz would like to know: what script this is, so that it can use script-specific shaping where necessary; what direction the script goes in; what language the text is written in. There are a lot of potential values here. Language should be one of the ISO639 language tags from <a href="http://www.microsoft.com/typography/otspec/languagetags.htm">here</a>; direction should be either <code>HB_DIRECTION_LTR</code>,
<code>HB_DIRECTION_RTL</code>, <code>HB_DIRECTION_TTB</code> (top to bottom), or <code>HB_DIRECTION_BTT</code>.

There are huge number of Harfbuzz scripts, but the one you're going to most use is "Latin" or <code>HB_SCRIPT_LATIN</code> if you want to pass that to <code>hb_buffer_set_script</code> directly instead of using <code>hb_tag_from_string</code>. For completeness, a full list of script strings and tags follows at the end of this post.

Now the buffer knows what it's dealing with. Let's get to the meat of the work: laying out the UTF-8 string into glyphs and then shaping those glyphs for a given font:

<pre>
    hb_buffer_add_utf8(buf, text, strlen(text), 0, strlen(text));
    hb_shape(hb_ft_font, buf, NULL, 0);
</pre>

Everything stays in the buffer, but we can extract it like so:

<pre>
    glyph_info   = hb_buffer_get_glyph_infos(buf, &glyph_count);
    glyph_pos    = hb_buffer_get_glyph_positions(buf, &glyph_count);
</pre>

<code>glyph_info</code> and <code>glyph_pos</code> are arrays of glyphs from 0 to <code>glyph_count</code>. The thing you'll want to get out of <code>glyph_info[i]</code> is the <code>codepoint</code> member, which is the glyph's ID in the font, which undoubtably you'll be passing to whatever is rendering this text. Now you also probably want to know how to render it: <code>glyph_pos</code> gives you <code>x_advance</code> and <code>y_advance</code>, which are how the rendering pen should move after rendering this glyph, and <code>x_offset</code> and <code>y_offset</code> which is where the glyph should be positioned relative to the pen. (usually zero) These are given in Freetype units, 64ths of a point.

If you need height and depth information for the glyph, then you need to go back to Freetype and ask it:

<pre>
void calculate_extents(box* b, hb_glyph_info_t glyph_info, hb_glyph_position_t glyph_pos, FT_Face ft_face) {
  const FT_Error error = FT_Load_Glyph(ft_face, glyph_info.codepoint, FT_LOAD_DEFAULT);
  if (error) return;

  const FT_Glyph_Metrics *ftmetrics = &ft_face->glyph->metrics;
  b->width = glyph_pos.x_advance /64.0;
  b->height = ftmetrics->horiBearingY / 64.0;
  b->depth = (ftmetrics->height - ftmetrics->horiBearingY) / 64.0;
}
</pre>

That's everything I needed to put text into glyphs and glyphs into boxes. The collected code can be found <a href="https://gist.github.com/simoncozens/6892796dd737212b0651">here</a>.

<hr>
And now, the Harfbuzz script list:

<small><pre class="no-code">Zyyy: HB_SCRIPT_COMMON
Zinh: HB_SCRIPT_INHERITED
Zzzz: HB_SCRIPT_UNKNOWN
Arab: HB_SCRIPT_ARABIC
Armn: HB_SCRIPT_ARMENIAN
Beng: HB_SCRIPT_BENGALI
Cyrl: HB_SCRIPT_CYRILLIC
Deva: HB_SCRIPT_DEVANAGARI
Geor: HB_SCRIPT_GEORGIAN
Grek: HB_SCRIPT_GREEK
Gujr: HB_SCRIPT_GUJARATI
Guru: HB_SCRIPT_GURMUKHI
Hang: HB_SCRIPT_HANGUL
Hani: HB_SCRIPT_HAN
Hebr: HB_SCRIPT_HEBREW
Hira: HB_SCRIPT_HIRAGANA
Knda: HB_SCRIPT_KANNADA
Kana: HB_SCRIPT_KATAKANA
Laoo: HB_SCRIPT_LAO
Latn: HB_SCRIPT_LATIN
Mlym: HB_SCRIPT_MALAYALAM
Orya: HB_SCRIPT_ORIYA
Taml: HB_SCRIPT_TAMIL
Telu: HB_SCRIPT_TELUGU
Thai: HB_SCRIPT_THAI
Tibt: HB_SCRIPT_TIBETAN
Bopo: HB_SCRIPT_BOPOMOFO
Brai: HB_SCRIPT_BRAILLE
Cans: HB_SCRIPT_CANADIAN_SYLLABICS
Cher: HB_SCRIPT_CHEROKEE
Ethi: HB_SCRIPT_ETHIOPIC
Khmr: HB_SCRIPT_KHMER
Mong: HB_SCRIPT_MONGOLIAN
Mymr: HB_SCRIPT_MYANMAR
Ogam: HB_SCRIPT_OGHAM
Runr: HB_SCRIPT_RUNIC
Sinh: HB_SCRIPT_SINHALA
Syrc: HB_SCRIPT_SYRIAC
Thaa: HB_SCRIPT_THAANA
Yiii: HB_SCRIPT_YI
Dsrt: HB_SCRIPT_DESERET
Goth: HB_SCRIPT_GOTHIC
Ital: HB_SCRIPT_OLD_ITALIC
Buhd: HB_SCRIPT_BUHID
Hano: HB_SCRIPT_HANUNOO
Tglg: HB_SCRIPT_TAGALOG
Tagb: HB_SCRIPT_TAGBANWA
Cprt: HB_SCRIPT_CYPRIOT
Limb: HB_SCRIPT_LIMBU
Linb: HB_SCRIPT_LINEAR_B
Osma: HB_SCRIPT_OSMANYA
Shaw: HB_SCRIPT_SHAVIAN
Tale: HB_SCRIPT_TAI_LE
Ugar: HB_SCRIPT_UGARITIC
Bugi: HB_SCRIPT_BUGINESE
Copt: HB_SCRIPT_COPTIC
Glag: HB_SCRIPT_GLAGOLITIC
Khar: HB_SCRIPT_KHAROSHTHI
Talu: HB_SCRIPT_NEW_TAI_LUE
Xpeo: HB_SCRIPT_OLD_PERSIAN
Sylo: HB_SCRIPT_SYLOTI_NAGRI
Tfng: HB_SCRIPT_TIFINAGH
Bali: HB_SCRIPT_BALINESE
Xsux: HB_SCRIPT_CUNEIFORM
Nkoo: HB_SCRIPT_NKO
Phag: HB_SCRIPT_PHAGS_PA
Phnx: HB_SCRIPT_PHOENICIAN
Cari: HB_SCRIPT_CARIAN
Cham: HB_SCRIPT_CHAM
Kali: HB_SCRIPT_KAYAH_LI
Lepc: HB_SCRIPT_LEPCHA
Lyci: HB_SCRIPT_LYCIAN
Lydi: HB_SCRIPT_LYDIAN
Olck: HB_SCRIPT_OL_CHIKI
Rjng: HB_SCRIPT_REJANG
Saur: HB_SCRIPT_SAURASHTRA
Sund: HB_SCRIPT_SUNDANESE
Vaii: HB_SCRIPT_VAI
Avst: HB_SCRIPT_AVESTAN
Bamu: HB_SCRIPT_BAMUM
Egyp: HB_SCRIPT_EGYPTIAN_HIEROGLYPHS
Armi: HB_SCRIPT_IMPERIAL_ARAMAIC
Phli: HB_SCRIPT_INSCRIPTIONAL_PAHLAVI
Prti: HB_SCRIPT_INSCRIPTIONAL_PARTHIAN
Java: HB_SCRIPT_JAVANESE
Kthi: HB_SCRIPT_KAITHI
Lisu: HB_SCRIPT_LISU
Mtei: HB_SCRIPT_MEETEI_MAYEK
Sarb: HB_SCRIPT_OLD_SOUTH_ARABIAN
Orkh: HB_SCRIPT_OLD_TURKIC
Samr: HB_SCRIPT_SAMARITAN
Lana: HB_SCRIPT_TAI_THAM
Tavt: HB_SCRIPT_TAI_VIET
Batk: HB_SCRIPT_BATAK
Brah: HB_SCRIPT_BRAHMI
Mand: HB_SCRIPT_MANDAIC
Cakm: HB_SCRIPT_CHAKMA
Merc: HB_SCRIPT_MEROITIC_CURSIVE
Mero: HB_SCRIPT_MEROITIC_HIEROGLYPHS
Plrd: HB_SCRIPT_MIAO
Shrd: HB_SCRIPT_SHARADA
Sora: HB_SCRIPT_SORA_SOMPENG
Takr: HB_SCRIPT_TAKRI
</pre></small>
