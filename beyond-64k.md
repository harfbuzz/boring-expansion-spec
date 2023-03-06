# Beyond 64K

This proposal enables fonts that have more than 65,535 glyphs.


## Design

We propose extending the number of glyphs in the font from 65,535 (16-bit) to 16,777,216 (24-bit).

Currently the number of glyphs in the font is encoded in the `maxp` table and is limited to 65535. In this proposal, the number of glyphs in the font will be calculated based on the length of the `loca` table. Moreover, tables that encode glyph indices as two-byte numbers are extended with versions of structures that encode 24-bit glyph indices.

While extending tables to encode 24-bit glyph indices, many tables are also extended to use 24-bit offsets instead of 16-bit offsets to allow for larger tables necessary for the larger number of glyphs.


## Per-table changes


### `maxp` table

The `numGlyphs` field of the `maxp` table for a font with more than 65,535 glyphs should be set to 65,535.


### `loca` / `glyf` tables

The `loca` / `glyf` tables are not required to have the same number of glyphs as specified in the `maxp` table. In fact, the length of the `loca` table now determines the number of glyphs in the font, which can be larger than `numGlyphs.maxp`. [issue](https://github.com/harfbuzz/boring-expansion-spec/issues/8)

Glyph table composites that need to access glyphs with glyph IDs larger than 65,535 will have to use the [VarComposites](https://github.com/harfbuzz/boring-expansion-spec/blob/main/glyf1.md) format which supports 24-bit GIDs. [issue](https://github.com/harfbuzz/boring-expansion-spec/issues/59)


### `cmap` table

The `cmap` formats 12 and 13 subtables already support 32-bit glyph indices.

A new sub-table format 16 will be added which is similar to format 14 but uses 24-bit glyph indices:
```
struct UVSMapping24 {
  uint24 unicodeValue;
  uint24 glyphID;
};
```
The rest of format 16 subtable is similar to format 14.

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/13)


### `hmtx` / `vmtx` tables

TODO


### `HVAR` / `VVAR` tables

TODO


### `VORG` table

TODO


### `COLR` table

TODO


### `sbix` table

The number of glyphs referenced by the table is the number of glyphs in the font as defined earlier. Same applies to non-OpenType tables `kerx` and `morx`.

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/44)


### `GDEF` / `GSUB` / `GPOS` tables

#### GSUB / GPOS version 2

The main `GSUB` / `GPOS` structs currently result in an offset-overflow with more than ~3k lookups. They are augmented with a version 2 that alleviates this problem:
```
struct GSUBGPOSVersion2 {
  Version version; // 0x00020000
  Offset24To<ScriptList> scripts;
  Offset24To<FeatureList> features;
  Offset24To<LookupList24> lookups;
  Offset32To<FeatureVariations> featureVars;
};
```
Note that the last item is a 32bit offset, for compatibility with version 0x00010001 tables.

were LookupList24 is a list16 of offset24 to Lookup structures:
```
using LookupList24 = List16OfOffsetTo<Lookup, uint24>;
```

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/58)
