# `VARC` table: Variable Composites / Components

## Variable Composite Description

A Variable Composite record is a concatenation of Variable Component records.
```
struct VarCompositeGlyph
{
  VarComponent components[];
};
```

## Variable Component Record

```
struct VarComponent
{
  uint16 flags;
  uint16 numAxes;
  variable
}
```

| type | name | notes |
|-|-|-|
| uint16 | flags | see below |
| uint8 | numAxes | Number of axes to follow |
| GlyphID16 or GlyphID24 | gid | This is a GlyphID16 if bit 0 of `flags` is clear, else GlyphID24 |
| uint8 or uint16 | axisIndices[numAxes] | This is a uint16 if bit 1 of `flags` is set, else a uint8 |
| F2DOT14 | axisValues[numAxes] | The axis value for each axis |
| uint32_t | axisValuesVarIndex | Optional, only present if bit 2 of `flags` is set |
| FWORD | TranslateX | Optional, only present if bit 3 of `flags` is set |
| FWORD |  TranslateY | Optional, only present if bit 4 of `flags` is set |
| F4DOT12 | Rotation | Optional, only present if bit 5 of `flags` is set |
| F6DOT10 | ScaleX | Optional, only present if bit 6 of `flags` is set |
| F6DOT10 | ScaleY | Optional, only present if bit 7 of `flags` is set |
| F4DOT12 | SkewX | Optional, only present if bit 8 of `flags` is set |
| F4DOT12 | SkewY | Optional, only present if bit 9 of `flags` is set |
| FWORD | TCenterX | Optional, only present if bit 10 of `flags` is set |
| FWORD |  TCenterY | Optional, only present if bit 11 of `flags` is set |
| uint32_t | transformVarIndex | Optional, only present if bit 12 of `flags` is set |

### Variable Component Flags

| bit number | meaning |
|-|-|
| 0 | gid is 24 bit |
| 1 | axis indices are shorts (clear = bytes, set = shorts) |
| 2 | axis values have variation |
| 3 | have TranslateX |
| 4 | have TranslateY |
| 5 | have Rotation |
| 6 | have ScaleX |
| 7 | have ScaleY |
| 8 | have SkewX |
| 9 | have SkewY |
| 10 | have TCenterX |
| 11 | have TCenterY |
| 12 | transformation fields have variation |
| 13 | reset unspecified axes |
| 14 | Use my metrics |
| 15 | Reserved. Set to 0 |

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
| ScaleY | ScaleX |
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
t = t.rotate(Rotation * math.pi)
t = t.scale(ScaleX, ScaleY)
t = t.skew(-SkewX * math.pi, SkewY * math.pi)
t = t.translate(-TCenterX, -TCenterY)
```

## `VARC` table
```
struct VARC
{
  uint16_t major; // 1
  uint16_t minor; // 0
  Offset32To<Coverage> coverage;
  Offset32To<MultiItemVariationStore>
  Offset32To<CFF2IndexOf<VarCompositeGlyph>> glyphRecords;
};

struct VarCompositeGlyphRecord
{
  VarComponentGlyphRecord[] components;
};
```

This design uses two new datastrucure: `MultiItemVariationStore`, and `CFF2IndexOf`. Let's look at those

- `CFF2IndexOf` is simply `CFFIndex` containing data of a particular type. `CFF2Index` is defined here:
   https://learn.microsoft.com/en-us/typography/opentype/spec/cff2#5-index-data

- `MultiIteVariationStore`: This is a new datastructure. It's a hybrid between
  `ItemVariationStore`, and `TupleVariationStore`, borring ideas (and data-structures)
  from both. Defined below:

## `MultiItemVariationStore`:

To be moved to [OpenType Font Variations Common Table Formats](https://learn.microsoft.com/en-us/typography/opentype/spec/otvarcommonformats).

A `MultiItemVariationStore` is a hybrid between `ItemVariationStore` and `TupleVariationStore`. Like `ItemVariationStore`, entries are addressed using a 32-bit `VarIdx`, with the top 16 bits called "outer" index, and lower 16 bits called the "inner" index.

Whereas the `ItemVariationStore` stores deltas for a single scalar value for each `VarIdx`, the `MultiItemVariationStore` stores deltas for a tuple for each `VarIdx`.

Compared to `TupleVariationStore`, the `MultiItemVariationStore` is optimized for smaller tuples and allows tuple-sharing, which is important for its efficiency over the `TupleVariationStore`. It also does not have some of the limitations that `TupleVariationStore` has, like the total size of an entry being limited to 64kb.

The top-level header of a `MultiItemVariationStore` is:
```
struct MultiItemVariationStore
{
  uint16 Format; // 1
  LOffsetTo<VarRegionList> VarRegionList;
  uint16_t MultiDataListCount
  Offset32To<MultiVariationData> MultiVariationData[MultiDataListCount];
};
```

```
struct MultiVariationData
{
  uint16 Format; // 1
  uint16 VarRegionCount;
  uint16 VarRegionIndex[VarRegionCount]
  CFF2IndexOf<TupleValues>
};
```
Where `TupleValues` is similar to the `gvar` [Packed Deltas](https://learn.microsoft.com/en-us/typography/opentype/spec/otvarcommonformats#packed-deltas) with a minor different that
if two top bits are both 1, then the values are 32bit. That difference should be incorporated in the Packed Deltas section of `gvar`, and it is backwards-compatible because the two top bits can currently never be on at the same time.


## Processing

The component glyphs to be loaded use the coordinate values specified (with any variations applied if present). For any unspecified axis, the value used depends on flag bit 13. If the flag is set, then the normalized value zero is used. If the flag is clear the axis values from current glyph being processed (which itself might recursively come from the font or its own parent glyphs) are used.  For example, if the font variations have `wght`=.25 (normalized), and current glyph being processed is using `wght`=.5 because it was referenced from another VarComposite glyph itself, when referring to a component that does _not_ specify the `wght` axis, if flag bit 13 is set, then the value of `wght`=0 (default) will be used. If flag bit 13 is clear, `wght`=.5 (from current glyph) will be used.

The component location and transform can vary. These variations are stored in the `MultiItemVariationStore` data-structure. The variations for location are referred to by the `axisValuesVarIndex` member of a component if any, and variations for transform are referred to by the `transformVarIndex` if any. For transform variations, only those fields specified and as such encoded as per the flags have variations.


**Note:** While it is the (undocumented?) behavior of the `glyf` table that glyphs loaded are shifted to align their LSB to that specified in the `hmtx` table, much like regular Composite glyphs, this does not apply to component glyphs being loaded as part of a variable-composite glyph.

**Note:** A static (non-variable) font that uses the `VARC` table, _would not_ have `fvar` / `avar` tables but _would_ have the `gvar` table in a TrueType-flavored font, or `CFF2` variations in a CFF-flavored font. This is because the components themselves store their variables in the classic way. The exception to this situation is a font with `VARC` table where does NOT vary the component locations and transforms, and does not encode any location for the components either. In practice, such a font would just use the `VARC` table like classic components.
