---
published: true
layout: post
title: Userspace and designspace - a note to self
---

Whenever I play with (or make) variable fonts, I get *hopelessly* confused about the difference between userspace and designspace coordinates. This is me trying to figure it out once and for all, and writing down what I find so that I won't get confused next time.

Variable fonts have axes, and those axes have points on them to determine the location. Those points have two different coordinate systems: userspace is, unsurprisingly, what you show to the user, and designspace is, also unsurprisingly, the one that the designer uses. Sometimes they're the same. But sometimes they're not. Sometimes you want to give the user a different range of values to select. I'm going to use [Nunito](https://fonts.google.com/specimen/Nunito) as my example for this piece.

Nunito's weight axis, in the design, is based on the stem width. So the light end, the axis minimum, is 42 because the stem is 42 units wide, while the heavy end, the axis maximum, is 208 units wide. These are the designspace coordinates.

But we want to show the user a more CSS-friendly weight axis, running from 200 to 1000. The sliders on our UI will run from 200 to 1000, not from 42 to 208. These are the userspace coordinates.

In the `.designspace` file, the axes are defined in *userspace* coordinates:

```xml
<axis tag="wght" name="Weight" minimum="200" maximum="1000" default="200"/>
```

But the *sources* - since they're an artefact of the design - are defined in *designspace* coordinates:

```xml
<source filename="Nunito-ExtraLight.ufo" name="Nunito ExtraLight" familyname="Nunito" stylename="ExtraLight">
    <location>
        <dimension name="Weight" xvalue="42"/>
        <dimension name="Italic" xvalue="0"/>
    </location>
</source>
```

The *instances* are *also* defined in designspace coordinates:

```xml
 <instance name="Nunito ExtraLight" familyname="Nunito" stylename="ExtraLight" filename="instance_ufo/Nunito-ExtraLight.ufo" stylemapfamilyname="Nunito ExtraLight" stylemapstylename="regular">
      <location>
        <dimension name="Weight" xvalue="42"/>
        <dimension name="Italic" xvalue="0"/>
      </location>
</instance>
```

In the final font, coordinates are either in userspace or they're normalized. In the binary `fvar` table, the axes are defined in *userspace* coordinates, so that's just like in the `.designspace` file:

```xml
    <Axis>
      <AxisTag>wght</AxisTag>
      <Flags>0x0</Flags>
      <MinValue>200.0</MinValue>
      <DefaultValue>200.0</DefaultValue>
      <MaxValue>1000.0</MaxValue>
  </Axis>
```

But the *instances* in the `fvar` table are *also* in userspace coordinates, which is *not* like the `.designspace` file:

```xml
    <!-- ExtraLight -->
    <!-- PostScript: Nunito-ExtraLight -->
    <NamedInstance flags="0x0" postscriptNameID="271" subfamilyNameID="258">
      <coord axis="wght" value="200.0"/>
      <coord axis="ital" value="0.0"/>
    </NamedInstance>
```

These too are values presented to users. User values, userspace.

Right. Now we get on to defining the relationship between userspace and designspace. (Or is it the other way around?) In the `.designspace` file, the `mapping` elements on an `axis` element map an `input` in *userspace* to an `output` in *designspace* (those attribute names aren't helpful, but it's too late now):

```xml
    <axis tag="wght" name="Weight" minimum="200" maximum="1000" default="200">
      <map input="200" output="42"/>
      <map input="300" output="61"/>
      <map input="400" output="81"/>
      <map input="600" output="101"/>
      <map input="700" output="125"/>
      <map input="800" output="151"/>
      <map input="900" output="178"/>
      <map input="1000" output="208"/>
    </axis>
```

Inside the binary font, this mapping is stored in the `avar` table, which looks like this:

```xml
    <segment axis="wght">
      <mapping from="-1.0" to="-1.0"/>
      <mapping from="0.0" to="0.0"/>
      <mapping from="0.125" to="0.11444"/>
      <mapping from="0.25" to="0.2349"/>
      <mapping from="0.5" to="0.3554"/>
      <mapping from="0.625" to="0.5"/>
      <mapping from="0.75" to="0.6566"/>
      <mapping from="0.875" to="0.8193"/>
      <mapping from="1.0" to="1.0"/>
    </segment>
```

Wait, what? OK, let's stop for a moment and go back to look at normalization. The sources are normalized. We see that the `gvar` table has the following locations for tuple variations:

```xml
        <coord axis="wght" min="0.0" value="0.5" max="1.0"/>
        <coord axis="wght" min="0.5" value="1.0" max="1.0"/>
```

(and then italic things.) These are the variations on the default master, so mentally insert another set of coordinates with `value="0.0"`. 

To get these values, we normalize the masters across *designspace*. 0 is the default value of the weight axis, so designspace=42; 1 is the maximum value of the weight axis, so designspace=208. The intermediate master is halfway between them, for `(42+208)/2=125`. And indeed that's just what we see in the `.designspace` file:

```xml
    <source filename="Nunito-M500.ufo" name="Nunito M500" familyname="Nunito" stylename="M500">
      <location>
        <dimension name="Weight" xvalue="125"/>
        <dimension name="Italic" xvalue="0"/>
      </location>
    </source>
```

(Ignore the "500", it's not helpful or accurate.)

So. To go from source location values to gvar tuple locations, you normalize them *across the range of the designspace*: `(125-42)/(208-42) = 0.5`. Masters are in designspace. Masters, designspace. User values, userspace.

So now the `avar` mapping. To create the `avar` table, normalize the userspace value across the userspace and map it to the designspace value normalized across the designspace.

```xml
<map input="200" output="42"/>
```

`(200-200)/(1000-200) = 0`, `(42-42)/(208-42) = 0`, and hence:

```xml
<mapping from="0.0" to="0.0"/>
```

And next:

```xml
<map input="300" output="61"/>
```

`(300-200)/(1000-200) = 0.125`, `(61-42)/(208-42) = 0.11445783`, and hence, with a bit of OpenType rounding:

```xml
<mapping from="0.125" to="0.11444"/>
```

And so on. Make sure that you add `avar` mappings for minimum, default and maximum, because they need to be there, and that's how you make an avar table.

*Summary*: `fvar` values all userspace. `gvar` values all normalized designspace. `avar` value map normalized designspace to normalized userspace. Man, that was like pulling teeth.

For bonus points: `fontTools.designspaceLib.Axis` has two functions, `map_forward` and `map_backward`. *Which way is which?*

```python
>>> from fontTools.designspaceLib import DesignSpaceDocument
>>> ds = DesignSpaceDocument.fromfile("master_ufo/Nunito.designspace")
>>> ds.axes[0].map_forward(200)
42.0
```

`map_forward` goes from *userspace* to *designspace*. `map_backward` goes from *designspace* to *userspace*. No, these names aren't helpful either.
