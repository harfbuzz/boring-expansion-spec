# Beyond 64K

This proposal enables fonts that have more than 65,535 glyphs.


## Design

We propose extending the number of glyphs in the font from 65,535 (16-bit) to 16,777,216 (24-bit).

Currently the number of glyphs in the font is encoded in the `maxp` table and is limited to 65535. In this proposal, the number of glyphs in the font will be calculated based on the length of the `loca` table. Moreover, tables that encode glyph indices as two-byte numbers are extended with versions of structures that encode 24-bit glyph indices.

While extending tables to encode 24-bit glyph indices, many tables are also extended to use 24-bit offsets instead of 16-bit offsets to allow for larger tables necessary for the larger number of glyphs.


## Per-table changes


### `maxp` table

The `numGlyphs` field of the `maxp` table for a font with more than 65,535 glyphs should be set to 65,535, although this is not required for the functioning of the design.


### `loca` / `glyf` tables

The `loca` / `glyf` tables are not required to have the same number of glyphs as specified in the `maxp` table. In fact, the length of the `loca` table now determines the number of glyphs in the font, which can be larger than `numGlyphs.maxp`. [issue](https://github.com/harfbuzz/boring-expansion-spec/issues/8)



#### Composite glyphs

Add a new flag to allow encoding 24bit glyph indices in composite glyphs:

```
enum ComponentGlyphFlags
    ARG_1_AND_2_ARE_WORDS       = 0x0001,
    ARGS_ARE_XY_VALUES          = 0x0002,
    ROUND_XY_TO_GRID            = 0x0004,
    WE_HAVE_A_SCALE             = 0x0008,
    MORE_COMPONENTS             = 0x0020,
    WE_HAVE_AN_X_AND_Y_SCALE    = 0x0040,
    WE_HAVE_A_TWO_BY_TWO        = 0x0080,
    WE_HAVE_INSTRUCTIONS        = 0x0100,
    USE_MY_METRICS              = 0x0200,
    OVERLAP_COMPOUND            = 0x0400,
    SCALED_COMPONENT_OFFSET     = 0x0800,
    UNSCALED_COMPONENT_OFFSET   = 0x1000,
    GID_IS_24BIT                = 0x2000
};
```

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/59)


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

Below we discuss `hmtx`. The case of `vmtx` is similar.

Currently the `hmtx` table length is determined by `hhea`/`maxp` in the following way:

* `hhea.metricDataFormat` is required to be 0 for the current format;
* `hmtx` is required to be `2 * hhea.numberOfHMetrics + 2 * maxp.numGlyphs` bytes long.

We describe how we to relax the second requirement, to allow encoding advance width of arbitrary number of glyphs in the `hmtx` table, regardless of the value of `maxp.numGlyphs`.

This is how an advance width is assigned to glyph indices beyond `maxp.numGlyphs`:

Let B be the excess bytes at the end of the `hmtx` table beyond the `2 * hhea.numberOfHMetrics + 2 * maxp.numGlyphs` bytes.

- If the length of B is odd, ignore the last byte of B.
- If B is empty, use the last advanceWidth in the `hmtx` table for all extra glyphs.
- Treat B as an array of `uint16` advance-width numbers of glyph indices starting at `maxp.numGlyphs`. For any glyph index that is in range in the font but out of range in this array, use the last item of the array.

No `lsb` (leftside-bearing) is encoded for glyphs beyond `maxp.numGlyphs`.

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/7)


#### `lsb` offsetting of glyphs

It is an undocumented behavior of OpenType that glyphs from the `glyf` table are shifted by `lsb - xMin` when rasterized. Since our extension to the `hmtx` table does _not_ encode `lsb` for new gids, no shifting is done for such glyphs. In other words, the `lsb` of such glyphs is assumed to be the same as `xMin`.


#### `tsb` and vertical glyphs

The vertical origin of glyphs in the `glyf` table is computed based on `tsb` encoded in the `vmtx` table. Since our extension to the `vmtx` table does not encode `tsb` for new glyphs, the `VORG` table _must_ be used to encode vertical glyph origins instead.

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/10)


### `HVAR` / `VVAR` tables

No changes needed.

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/12)


### `VORG` table

Currently the length of the `VORG` table is determined from its major/minor version and the subsequent content. The only currently defined version 1.0 can only encode 16bit glyph indices. Here is how we expand the table's definition to encode glyph vertical origin for glyph indices beyond `maxp.numGlyphs`:

Derive the length of the table from the version number and the table struct, then derive the excess bytes, then follow the same algorithm for deriving advance-width in the hmtx/vmtx expansion and use those numbers as vertical origin instead.

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/11)


### `COLR` table

To be done as part of `COLRv2`.

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/47)


### `sbix` table

The number of glyphs referenced by the table is the number of glyphs in the font as defined earlier. Same applies to non-OpenType tables `kerx` and `morx`.

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/44)


### `GDEF` / `GSUB` / `GPOS` tables

#### `GDEF` version 2

The main `GDEF` struct is augmented with a version 2 to alleviate offset-overflows when classDef and other structs grow large:
```
struct GDEFVersion2 {
  Version version; // 0x00020000
  Offset24To<ClassDef> glyphClassDef;
  Offset24To<AttachList> attachList;
  Offset24To<LigCaretList> ligCaretList;
  Offset24To<ClassDef> markAttachClassDef;
  Offset24To<MarkGlyphSets> markGlyphSets;
  Offset32To<ItemVariationStore> varStore;
};
```
Note that the varStore offset is 32bit, for compatibility with 0x00010003 version.

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/36)


#### `GSUB`

The following sections when combined, fully cover the `GSUB` table extension:
* `GSUB` / `GPOS` version 2
* `Coverage` / `ClassDef` formats 3 & 4
* `GSUB` `SingleSubst` formats 3 & 4
* `GSUB` `MultipleSubst` / `AlternateSubst` format 2
* `GSUB` `LigatureSubst` format 2
* `GSUB` / `GPOS` `(Chain)Context` format 4 & 5

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/35)


#### `GPOS`

The following sections when combined, fully cover the `GSUB` table extension:
* `GSUB` / `GPOS` version 2
* `Coverage` / `ClassDef` formats 3 & 4
* `GPOS` `PairPos` formats 3 & 4
* `GPOS` `MarkBasePos` / `MarkLigPos` / `MarkMarkPos` format 2
* `GSUB` / `GPOS` `(Chain)Context` format 4 & 5

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/39)


#### `GSUB` / `GPOS` version 2

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

`LookupList24` is a `List16` of `Offset24` to `Lookup` structures:
```
using LookupList24 = List16OfOffse24tTo<Lookup>;
```

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/58)


#### `Coverage` / `ClassDef` formats 3 & 4

Coverage and ClassDef tables are augmented with formats 3 and 4 to allow for gid24, which parallel formats 1 & 2 respectively:
```
struct CoverageFormat3 {
  uint16 format; // == 3
  Array16Of<GlyphID24> glyphs;
};

struct CoverageFormat4 {
  uint16 format; // == 4
  Array16Of<Range24Record> ranges;
};

struct ClassDefFormat3 {
  uint16 format; // == 3
  GlyphID24 startGlyphID;
  Array24Of<uint16> classes;
};

struct ClassDefFormat4 {
  uint16 format; // == 4
  Array24Of<Range24Record> ranges;
};

struct Range24Record {
  GlyphID24 startGlyphID;
  GlyphID24 endGlyphID;
  uint16 value;
};
```

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/30)


#### `GSUB` `SingleSubst` formats 3 & 4

Clarify that in format 1, delta addition math only affects the lower 16bits of the gid. Format 3 delta addition math is module 2^24.

Two new formats, 3 & 4, are introduced, that parallel formats 1 & 2 respectively:
```
struct SingleSubstFormat3 {
  uint16 format; // == 3
  Offset24To<Coverage> coverage;
  int24 deltaGlyphID;
};

struct SingleSubstFormat4 {
  uint16 format; // == 4
  Offset24To<Coverage> coverage;
  Array16Of<GlyphID24> substitutes;
};
```

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/31)


#### `GSUB` `MultipleSubst` / `AlternateSubst` format 2

Format 2 is introduced to enable gid24:
```
struct MultipleSubstFormat2 {
  uint16 format; // == 2
  Offset24To<Coverage> coverage;
  Array16Of<Offset24To<Sequence24>> sequences;
};

typedef Array16Of<GlyphID24> Sequence24;

struct AlternateSubstFormat2 {
  uint16 format; // == 2
  Offset24To<Coverage> coverage;
  Array16Of<Offset24To<AlternateSet24>> alternateSets;
};

typedef Array16Of<GlyphID24> AlternateSet24;
```

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/32)


#### `GSUB` `LigatureSubst` format 2

Format 2 is introduced to enable gid24:
```
struct LigatureSubstFormat2 {
  uint16 format; == 2
  Offset24To<Coverage> coverage;
  Array16Of<Offset24To<LigatureSet24>> ligatureSets;
};

struct LigatureSet24 {
  Array16Of<Offset16To<Ligature24>> ligatures;
};

struct Ligature24 {
  GlyphID24 ligatureGlyph;
  uint16 componentCount;
  GlyphID24 componentGlyphIDs[componentCount - 1];
};
```

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/33)


#### `GPOS` `PairPos` formats 3 & 4

Two new formats, 3 & 4, are introduced that parallel formats 1 & 2.

Format 4 is only introduced to alleviate offset-overflow issues and is not otherwise needed for gid24 support.

```
struct PosFormat3 {
  uint16 format; == 3
  Offset24To<Coverage> coverage;
  uint16 valueFormat1;
  uint16 valueFormat2;
  Array16Of<Offset24To<PairSet24>> pairSets;
};

struct PairSet24 {
  uint24 pairValueCount;
  PairValueRecord24 pairValueRecords[pairValueCount];
};

struct PairValueRecord24 {
  GlyphID24 secondGlyph;
  ValueRecord valueRecord1;
  ValueRecord valueRecord2;
};
```

```
struct PosFormat4 {
  uint16 format; == 4
  Offset24To<Coverage> coverage;
  uint16 valueFormat1;
  uint16 valueFormat2;
  Offset24To<ClassDef> classDef1;  
  Offset24To<ClassDef> classDef2;
  uint16 class1Count;
  uint16 class2Count;
  Class1Record class1Records[class1Count];
};
```

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/38)


#### `GPOS` `MarkBasePos` / `MarkLigPos` / `MarkMarkPos` format 2

Format 2 is introduce just to alleviate offset-overflow issues at the top-level structure. All downstream structures are reused:
```

The Coverage table parts are covered in #30.

Add one new format of each, just to upgrade offsets of the top-level subtable to 24bit. All downstream structs are reused and not expanded.

struct MarkBasePosFormat2 {
  uint16 format; // == 2
  Offset24To<Coverage> markCoverage;
  Offset24To<Coverage> baseCoverage;
  uint16 markClassCount;
  Offset24To<MarkArray> markArray;
  Offset24To<BaseArray> baseArray;
};

struct MarkLigaturePosFormat2 {
  uint16 format; // == 2
  Offset24To<Coverage> markCoverage;
  Offset24To<Coverage> ligatureCoverage;
  uint16 markClassCount;
  Offset24To<MarkArray> markArray;
  Offset24To<LigatureArray> ligatureArray;
};

struct MarkMarkPosFormat2 {
  uint16 format; // == 2
  Offset24To<Coverage> mark1Coverage;
  Offset24To<Coverage> mark2Coverage;
  uint16 markClassCount;
  Offset24To<MarkArray> mark1Array;
  Offset24To<Mark2Array> mark2Array;
};
```

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/40)


#### `GSUB` / `GPOS` `(Chain)Context` format 4 & 5

Add `Context` and `ChainContext` format 4 that parallels format 1 for gid24:
```
struct ContextFormat4 {
  uint16 format; == 4
  Offset24To<Coverage> coverage;
  Array16Of<Offset24To<GlyphRuleSet24>> ruleSets;
};

struct GlyphRuleSet24 {
  Array16Of<Offset16To<GlyphRule24>> rules;
};

struct GlyphRule24 {
  uint16 glyphCount;
  GlyphID24 glyphs[inputGlyphCount - 1];
  uint16 seqLookupCount;
  SequenceLookupRecord seqLookupRecords[seqLookupCount];
};
```
```
struct ChainContextFormat4 {
  uint16 format; == 4
  Offset24To<Coverage> coverage;
  Array16Of<Offset24To<ChainGlyphRuleSet24>> ruleSets;
};

struct ChainGlyphRuleSet24 {
  Array16Of<Offset16To<ChainGlyphRule24>> rules;
};

struct ChainGlyphRule24 {
  uint16 backtrackGlyphCount; 
  GlyphID24 backtrackGlyphs[backtrackGlyphCount];
  uint16 inputGlyphCount;
  GlyphID24 inputGlyphs[inputGlyphCount - 1];
  uint16 lookaheadGlyphCount;
  GlyphID24 lookaheadGlyphs[lookaheadGlyphCount];
  uint16 seqLookupCount;
  SequenceLookupRecord seqLookupRecords[seqLookupCount];
};
```

Add `Context` and `ChainContext` format 5 that parallels format 2 for offset-overflow alleviation:
```
struct ContextFormat5 {
  uint16 format; == 5
  Offset24To<Coverage> coverage;
  Offset24To<ClassDef> classDef;
  Array16Of<Offset24To<ClassRuleSet>> ruleSets;
};
```
```
struct ChainContextFormat5 {
  uint16 format; == 5
  Offset24To<Coverage> coverage;
  Offset24To<ClassDef> backtrackClassDef;
  Offset24To<ClassDef> inputClassDef;
  Offset24To<ClassDef> lookaheadClassDef;
  Array16Of<Offset24To<ClassRuleSet>> ruleSets;
};
```
The `RuleSet` and `ChainRuleSet` are _not_ extended, because they are class-based, not glyph-based, so no extension is necessary.

Format 3 (Coverage-based format) is _not_ extended, because it only encodes one rule, so overflows are unlikely.

[issue](https://github.com/harfbuzz/boring-expansion-spec/issues/34)
