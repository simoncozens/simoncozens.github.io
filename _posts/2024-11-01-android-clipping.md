---
published: true
layout: post
title: "Android Text Clipping"
---

Recently I had to understand how Android interprets a font's vertical metrics, and when there will be or will not be clipping in a text box. And the answer is: "it depends".

Boy, does it depend.

First, it depends on what technology stack is being used. Some Android applications are written in [Flutter](https://flutter.dev) - this is Google's new cross-platform mobile (and web) development framework. The majority of apps, however, are written in the "classic" Android technologies, either in Java or Kotlin, and based on a framework called Views. More recently, [Jetpack Compose](https://developer.android.com/compose) is a framework which sits on top of Views and makes app development a bit easier.

So here's our first "it depends": Flutter-based or Views-based? Let's get Flutter out of the way first because it's an easy one: In text, Flutter is rendered with *completely different* code to Views and Compose. This is actually a good thing because as far as I can tell, text doesn't clip to *any* vertical metrics in Flutter. Go as tall or as deep as you like!

Now comes our second "it depends": Compose or Views? Compose clipping depends a little on the widgets in question. Let me just define some quick terminology here because it'll be useful later:

* The *font ascent* is the value of `sTypoAscender` in the font's `OS/2` table if the fsSelection bit 7 (`USE_TYPO_METRICS`) bit is set and the value of `ascender` in the `hhea` table if it isn't.
* The *font descent* is the value of `sTypoDescender` in the font's `OS/2` table if `USE_TYPO_METRICS` is set and the value of `descender` in the `hhea` table if it isn't.

So: A Compose `OutlinedTextField` clips text to the font ascent and descent. A Compose `Text` does not clip, but its line height is determined by the font ascent and descent. A Compose `OutlinedButton` clips to a little above and below the ascent and descent. (I'm not sure how much "a little" is.)

All right, into Views. An Android `TextView` has a number of flags which control its behaviour. We will consider two for now (it will get worse, I promise): [`includeFontPadding`](https://developer.android.com/reference/android/widget/TextView#attr_android:includeFontPadding) and [`elegantTextHeight`](https://developer.android.com/reference/android/widget/TextView#attr_android:fallbackLineSpacing).

* If `elegantTextHeight` is set to false and `includeFontPadding` is set to false, text is clipped to font ascent and descent. Phew.
* If `elegantTextHeight` if false and `includeFontPadding` is `true`, then the line height is set (and text is clipped) to *either* the font ascent, or the heighest `yMax` value of the `glyf` table bounding box of all the glyphs in the run (*not* the shaped position of the glyph!), whichever is the higher. And likewise, the font depth is set to the minimum of the font descender and the `glyf` table `yMin` value of the glyphs in the run.

Are you ready for this?

If `elegantTextHeight` is set to true then *the vertical metrics in the font are completely ignored* and replaced by [hard-coded constants deep in the Android core graphics library](https://android.googlesource.com/platform/frameworks/base/+/ee699a6/core/jni/android/graphics/Paint.cpp#399). With the effect that:

* If `elegantTextHeight` is true, and `includeFontPadding` is false, text is clipped to `1900/2048` multiplied by the font's UPM at the top and `-500/2048` multiplied by the font's UPM at the bottom.
* If `elegantTextHeight` is true, and `includeFontPadding` is true, text is clipped to `2500/2048` multiplied by the font's UPM at the top and `-1000/2048` multiplied by the font's UPM at the bottom.

Oh, and which of these flags is on or off by default depends on Android (and Comppose, if your app is using that) version. In Android 15, [elegantTextHeight was turned on by default](https://developer.android.com/about/versions/15/behavior-changes-15#elegant-text-height). In *Compose* 1.2.0-alpha05, [includeFontPadding was turned off by default](https://developer.android.com/jetpack/androidx/releases/compose-ui#1.2.0-alpha05) *for Compose widgets*; in 1.2.0-beta01, [it was turned on by default](https://developer.android.com/jetpack/androidx/releases/compose-ui#1.2.0-beta01).

All right, now the kicker.

All that I have told you is true *for an individual font*. But if you are using multiple fonts in the context of a fallback stack, which you usually are, which set of metrics are used depends entirely on the value of *another* flag: [`fallbackLineSpacing`](https://developer.android.com/reference/android/widget/TextView#attr_android:fallbackLineSpacing).

* If `fallbackLineSpacing` is false, the algorithm above is used for the *top* font in the stack, regardless of which font is used to render glyphs in the run.
* If `fallbackLineSpacing` is true, the algorithm above is used for the *used* font in the stack.

Apparently `fallbackLineSpacing` is true by default.

> I made a custom font and a bunch of Android apps (and spent much too long reading Android's source code) to discover all this...

![](https://simoncozens.github.io/images/android.png)
