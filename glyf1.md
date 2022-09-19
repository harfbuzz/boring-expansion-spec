# `glyf`—Glyf Data Table Version 1 / Variable Components

The glyf data table ([`glyf`](https://docs.microsoft.com/en-us/typography/opentype/spec/glyf)) stores the glyph outline data.  The version number for this table is stored in the [`head`](https://docs.microsoft.com/en-us/typography/opentype/spec/head) table's `glyphDataFormat` field.  The only currently defined version of the `glyf` table is version 0.  In this proposal we propose version 1 of the `glyf` table to add the following features:

* **Variable Composites / Components:** This proposal is mainly based on BlackFoundry's [Variable Components](https://github.com/BlackFoundryCom/variable-components-spec) and [Storing Variable Components in UFO files](https://github.com/BlackFoundryCom/variable-components-in-ufo).

In `glyf` table format 0, there exist two types of glyphs: Simple glyphs which have a `numberOfContours` of >= 0, and Composite glyphs that have a `numberOfContours` of < 0, with a recommended value of -1.

In `glyf` table format 1, the old-style Simple and Composite glyphs exist as well. The old-style Composite glyphs shall always use a `numberOfContours` of -1. A new composite glyph type call VariableComposite will be introduced which uses `numberOfContours` of -2. A VariableComposite glyph has the ability to refer to _variable components_.


## Variable Composite Description

A Variable Composite glyph starts with the standard glyph header with a `numberOfContours` of -2, followed by a number of Variable Component records.


## Variable Component Record




