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
| uint8 or uint16 | numAxes | This is a uint16 if bit 3 of `flags` is set, else a uint8 |
| uint8 or uint16 | axisIndices[numAxes] | This is a uint16 if bit 3 of `flags` is set, else a uint8<br/>The most significant bit of each axisIndex tells whether this axis has a VarIdx in the VarIdxs array below. Bits 0..6 (uint8) or 0..14 (uint16) form the axis index. |
| Coord16 | axisValues[numAxes] | The axis value for each axis |
| Angle16 | Rotation | Optional, only present if it 5 of `flags` is set |
| Scale16 | ScaleX | Optional, only present if it 6 of `flags` is set |
| Scale16 | ScaleY | Optional, only present if it 7 of `flags` is set |
| Angle16 | SkewX | Optional, only present if it 8 of `flags` is set |
| Angle16 | SkewY | Optional, only present if it 9 of `flags` is set |
| Int16 | TCenterX | Optional, only present if it 10 of `flags` is set |
| int16 |  TCenterY | Optional, only present if it 11 of `flags` is set |


### Transformation

The transformation data consists of individual optional fields, which can be
used to construct a transformation matrix.

Transformation fields:

| name | default value |
|-|-|
| Rotation | 0 |
| ScaleX | 1 |
| ScaleY | 1 |
| SkewX | 0 |
| SkewY | 0 |
| TCenterX | 0 |
| TCenterY | 0 |

The `TCenterX` and `TCenterY` values represent the “center of transformation”.
This is separate from the component offset as stored in the `glyf` table.

Details of how to build a transformation matrix, as pseudo-Python/fontTools
code, where `(X, Y)` is the component offset from the `glyf` table:

```python
# Using fontTools.misc.transform.Transform
t = Transform()  # Identity
t = t.translate(X + TCenterX, Y + TCenterY)
t = t.rotate(Rotation)
t = t.scale(ScaleX, ScaleY)
t = t.skew(SkewX, SkewY)
t = t.translate(-TCenterX, -TCenterX)
```

Component flags:

| bit number | meaning |
|-|-|
| 0..2 | Number of integer bits for ScaleX and ScaleY, mask: 0x07 |
| 3 | axis indices are shorts (clear = bytes, set = shorts) |
| 4 | Transformation fields have VarIdx |
| 5 | have Rotation |
| 6 | have ScaleX |
| 7 | have ScaleY |
| 8 | have SkewX |
| 9 | have SkewY |
| 10 | have TCenterX |
| 11 | have TCenterY |
| 12 | If ScaleY is missing: take value from ScaleX (to be discussed here: https://github.com/BlackFoundryCom/variable-components-spec/issues/2) |
| 13 | (reserved, set to 0) |
| 14 | (reserved, set to 0) |
| 15 | (reserved, set to 0) |




