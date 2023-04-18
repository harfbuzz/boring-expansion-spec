# `glyf`—Glyf Data Table Format 1

The glyf data table ([`glyf`](https://docs.microsoft.com/en-us/typography/opentype/spec/glyf)) stores the glyph outline data.  The format number for this table is stored in the [`head`](https://docs.microsoft.com/en-us/typography/opentype/spec/head) table's `glyphDataFormat` field.  The only currently defined format of the `glyf` table is format 0.  In this proposal we propose format 1 of the `glyf` table to add the following features:

* **Cubic-Bezier Outlines:** [Proposal](glyf1-cubicOutlines.md).
* **Variable Composites / Components:** [Proposal](glyf1-varComposites.md).
