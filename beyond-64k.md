# Beyond 64K

This proposal enables fonts that have more than 65,535 glyphs.

## Design

We propose extending the number of glyphs in the font from 65,535 (16-bit) to 16,777,216 (24-bit).

Currently the number of glyphs in the font is encoded in the `maxp` table and is limited to 65535. In this proposal, the number of glyphs in the font will be calculated based on the length of the `loca` table. Moreover, tables that encode glyph indices as two-byte numbers are extended with versions of structures that encode 24-bit glyph indices.

While extending tables to encode 24-bit glyph indices, many tables are also extended to use 24-bit offsets instead of 16-bit offsets to allow for larger tables necessary for the larger number of glyphs.

## maxp table

The `numGlyphs` field of the `maxp` table for a font with more than 65,535 glyphs should be set to 65,535.

## loca / glyf table

The `loca` / `glyf` tables are not required to have the same number of glyphs as specified in the `maxp` table. In fact, the length of the `loca` table now determines the number of glyphs in the font, which can be larger than `numGlyphs.maxp`.

Glyph table composites that need to access glyphs with glyph IDs larger than 65,535 will have to use the [VarComposites](https://github.com/harfbuzz/boring-expansion-spec/blob/main/glyf1.md) format which supports 24-bit GIDs.

## cmap table

## hmtx / vmtx table


## VORG table

## COLR table

## GDEF / GSUB / GPOS

