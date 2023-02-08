# `glyf`—Glyf Data Table Version 1: Cubic-Bezier Outlines

The glyf data table ([`glyf`](https://docs.microsoft.com/en-us/typography/opentype/spec/glyf)) stores the glyph outline data.  The version number for this table is stored in the [`head`](https://docs.microsoft.com/en-us/typography/opentype/spec/head) table's `glyphDataFormat` field.  The only currently defined version of the `glyf` table is version 0. We propose version 1 of the `glyf` table to add multiple features. See full [proposal](glyf1.md) for all features. In this proposal we cover: _Cubic-Bezier Outlines_.

In `glyf` table format 0, glyph outlines use quadratic Bezier curve segments.

In `glyf` table format 1, glyph outlines can mix quadratic and cubic Bezier curve segments.

## Specification

Add the following flag to the [Simple Glyph Description](https://learn.microsoft.com/en-us/typography/opentype/spec/glyf#simple-glyph-description) section's _Simple Glyph Flags_:
| Mask | Name  | Description |
|------|-------|-------------|
| 0x80 | CUBIC | Bit 7: Off-curve point belongs to a cubic-Bezier segment |

The number of consecutive cubic off-curve points _must_ be even.

The CUBIC flag _must_ only be used on off-curve points. It is _reserved_ and must be set to zero, for on-curve points.

## Processing

Every successive two off-curve points that have the CUBIC bit set define a cubic Bezier segment, and an implied on-curve point is inserted at the mid-point of the second off-curve point of a segment and the first off-curve point of the next segment.

TODO: Fully specify implied on-curve points.

TODO: Expand
