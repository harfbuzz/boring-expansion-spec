# `glyf`—Glyf Data Table Version 1 / Variable Components

The glyf data table ([`glyf`](https://docs.microsoft.com/en-us/typography/opentype/spec/glyf)) stores the glyph outline data.  The version number for this table is stored in the [`head`](https://docs.microsoft.com/en-us/typography/opentype/spec/head) table's `glyphDataFormat` field.  The only currently defined version of the `glyf` table is version 0.  In this proposal we propose version 1 of the `glyf` table to add the following features:

* **Variable Composites / Components:** This proposal is mainly based on BlackFoundry's [Variable Components](https://github.com/BlackFoundryCom/variable-components-spec) and [Storing Variable Components in UFO files](https://github.com/BlackFoundryCom/variable-components-in-ufo).

In `glyf` table format 0, there exist two types of glyphs: Simple glyphs which have a `numberOfContours` of >= 0, and Composite glyphs that have a `numberOfContours` of < 0, with a recommended value of -1.

In `glyf` table format 1, the old-style Simple and Composite glyphs exist as well. The old-style Composite glyphs shall always use a `numberOfContours` of -1. A new composite glyph type call VariableComposite will be introduced which uses `numberOfContours` of -2. A VariableComposite glyph has the ability to refer to _variable components_.


## Variable Composite Description

A Variable Composite glyph starts with the standard glyph header with a `numberOfContours` of -2, followed by a number of Variable Component records.


## Variable Component Record


| type | name | notes |
|-|-|-|
| uint16 | flags | see below |
| GlyphID16 or GlyphID24 | gid | This is a GlyphID16 if bit 12 of `flags` is clear, else GlyphID24 |
| uint8 | numAxes | Number of axes to follow |
| uint8 or uint16 | axisIndices[numAxes] | This is a uint16 if bit 1 of `flags` is set, else a uint8 |
| F2DOT14 | axisValues[numAxes] | The axis value for each axis |
| FWORD | TranslateX | Optional, only present if it 3 of `flags` is set |
| FWORD |  TranslateY | Optional, only present if it 4 of `flags` is set |
| F4DOT12 | Rotation | Optional, only present if it 5 of `flags` is set |
| F6DOT10 | ScaleX | Optional, only present if it 6 of `flags` is set |
| F6DOT10 | ScaleY | Optional, only present if it 7 of `flags` is set |
| F4DOT12 | SkewX | Optional, only present if it 8 of `flags` is set |
| F4DOT12 | SkewY | Optional, only present if it 9 of `flags` is set |
| FWORD | TCenterX | Optional, only present if it 10 of `flags` is set |
| FWORD |  TCenterY | Optional, only present if it 11 of `flags` is set |


### Variable Component Transformation

The transformation data consists of individual optional fields, which can be
used to construct a transformation matrix.

Transformation fields:

| name | default value |
|-|-|
| TranslateX | 0 |
| TranslateY | 0 |
| Rotation | 0 |
| ScaleX | 1 |
| ScaleY | 1 |
| SkewX | 0 |
| SkewY | 0 |
| TCenterX | 0 |
| TCenterY | 0 |

The `TCenterX` and `TCenterY` values represent the “center of transformation”.

Details of how to build a transformation matrix, as pseudo-Python code:

```python
# Using fontTools.misc.transform.Transform
t = Transform()  # Identity
t = t.translate(TranslateX + TCenterX, TranslateY + TCenterY)
t = t.rotate(Rotation)
t = t.scale(ScaleX, ScaleY)
t = t.skew(-SkewX, SkewY)
t = t.translate(-TCenterX, -TCenterX)
```

### Variable Component Flags

| bit number | meaning |
|-|-|
| 0 | USE_MY_METRICS |
| 1 | axis indices are shorts (clear = bytes, set = shorts) |
| 2 | If ScaleY is missing: take value from ScaleX (to be discussed here: https://github.com/BlackFoundryCom/variable-components-spec/issues/2) |
| 3 | have TranslateX |
| 4 | have TranslateY |
| 5 | have Rotation |
| 6 | have ScaleX |
| 7 | have ScaleY |
| 8 | have SkewX |
| 9 | have SkewY |
| 10 | have TCenterX |
| 11 | have TCenterY |
| 12 | gid is 24 bit |
| 13 | axis indices have variation |
| 14 | (reserved, set to 0) |
| 15 | (reserved, set to 0) |

## Processing

Variations of component records are processed this way: For each composite record, a vector of coordinate points is prepared, and its variation applied from the `gvar` table. The coordinate points are a concatenation of those for each variable component in order.

The coordinate points for each variable component consist of those of each axis value if flag bit 13 is set, represented as the X value of a coordinate point; followed by up to five points representing the transformation (regardless of which transformation components are encoded according to the flags). The five points encode, in order, in their X,Y components, the following transformation components: (`TranslateX`,`TranslateY`), (`Rotation`,0), (`ScaleX`,`ScaleY`), (`SkewX`,`SkewY`), (`TCenterX`,`TCenterY`). Only the transformation components present according to the flag bits are encoded.


