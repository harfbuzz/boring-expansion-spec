# `glyf`—Glyf Data Table Format 1: Cubic-Bezier Outlines

The glyf data table ([`glyf`](https://docs.microsoft.com/en-us/typography/opentype/spec/glyf)) stores the glyph outline data.  The format number for this table is stored in the [`head`](https://docs.microsoft.com/en-us/typography/opentype/spec/head) table's `glyphDataFormat` field.  The only currently defined format of the `glyf` table is format 0. We propose format 1 of the `glyf` table to add multiple features. See full [proposal](glyf1.md) for all features. In this proposal we cover: _Cubic-Bezier Outlines_.

In `glyf` table format 0, glyph outlines use quadratic Bezier curve segments.

In `glyf` table format 1, glyph outlines can mix quadratic and cubic Bezier curve segments.

## Specification

Add the following flag to the [Simple Glyph Description](https://learn.microsoft.com/en-us/typography/opentype/spec/glyf#simple-glyph-description) section's _Simple Glyph Flags_:
| Mask | Name  | Description |
|------|-------|-------------|
| 0x80 | `CUBIC` | Bit 7: Off-curve point belongs to a cubic-Bezier segment |

### Restrictions

Currently, there are several restrictions on how the `CUBIC` flag can be used. If any of the conditions below are not met, the behavior is undefined.

The number of consecutive cubic off-curve points within a contour (without wrap-around) _must_ be even.

All the off-curve points between two on-curve points (with wrap-around) `must` either have the `CUBIC` flag clear, or have the `CUBIC` flag set.

The `CUBIC` flag _must_ only be used on off-curve points. It is _reserved_ and _must_ be set to zero, for on-curve points.

## Processing

Every successive two off-curve points that have the `CUBIC` bit set define a cubic Bezier segment. Within any consecutive set of cubic off-curve points within a contour (with wrap-around), an implied on-curve point is inserted at the mid-point between every second off-curve point and the next one.

If there are no on-curve points and all (even number of) off-curve points are `CUBIC`, the first off-curve point is considered the first control-point of a cubic-Bezier, and implied on-curve points are inserted between the every second point and the next one as usual.

As in the `glyf` version 0 table, an implied on-curve point is inserted between any two neighboring quadratic off-curve points.
