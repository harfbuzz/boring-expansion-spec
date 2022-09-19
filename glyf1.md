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



