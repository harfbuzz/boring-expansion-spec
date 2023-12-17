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
| uint32_t | axisValuesVarIdx | Optional, only present if bit 2 of `flags` is set |
| FWORD | TranslateX | Optional, only present if bit 3 of `flags` is set |
| FWORD |  TranslateY | Optional, only present if bit 4 of `flags` is set |
| F4DOT12 | Rotation | Optional, only present if bit 5 of `flags` is set |
| F6DOT10 | ScaleX | Optional, only present if bit 6 of `flags` is set |
| F6DOT10 | ScaleY | Optional, only present if bit 7 of `flags` is set |
| F4DOT12 | SkewX | Optional, only present if bit 8 of `flags` is set |
| F4DOT12 | SkewY | Optional, only present if bit 9 of `flags` is set |
| FWORD | TCenterX | Optional, only present if bit 10 of `flags` is set |
| FWORD |  TCenterY | Optional, only present if bit 11 of `flags` is set |
| uint32_t | transformVarIdx | Optional, only present if bit 12 of `flags` is set |

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

The top-level header of a `MultiItemVariationStore` is:
```
struct MultiItemVariationStore
{
  uint16 Format; // 1
  LOffsetTo<VarRegionList> VarRegionList;
  uint16_t MultiDataListCount
  Offset32To<MultiVariationData> MultiVariationData;
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
Where `TupleValues` is similar to https://learn.microsoft.com/en-us/typography/opentype/spec/otvarcommonformats#packed-deltas with a minor different that
if two top bits are both 1, then the values are 32bit.


## Processing

Variations of composite glyphs are processed this way: For each composite glyph, a vector of coordinate points is prepared, and its variation applied from the `gvar` table. The coordinate points for the composite glyph are a concatenation of those for each variable component in order. For the purposes of `gvar` delta IUP calculations, each point is considered to be in its own contour.

The coordinate points for each variable component consist of those for each axis value if flag bit 13 is set, represented as the X value of a coordinate point; followed by up to five points representing the transformation. The five possible points encode, in order, in their X,Y components, the following transformation components: (`TranslateX`,`TranslateY`), (`Rotation`,0), (`ScaleX`,`ScaleY`), (`SkewX`,`SkewY`), (`TCenterX`,`TCenterY`). Only the transformation components present according to the flag bits are encoded. For example, the point (`TranslateX`,`TranslateY`) is encoded if and only if either of flag 3 or 4 is set. If that point is encoded, the variations from it will be applied to the transformation components even if a specific component has its flag bit off. For example, if only flag 3 is set (`TranslateX`), any variations for `TranslateY` are still applied to the `TranslateY` of the transformation.

The component glyphs to be loaded use the coordinate values specified (with any variations applied if present). For any unspecified axis, the value used depends on flag bit 14. If the flag is set, then the normalized value zero is used. If the flag is clear the axis values from current glyph being processed (which itself might recursively come from the font or its own parent glyphs) are used.  For example, if the font variations have `wght`=.25 (normalized), and current glyph being processed is using `wght`=.5 because it was referenced from another VarComposite glyph itself, when referring to a component that does _not_ specify the `wght` axis, if flag bit 14 is set, then the value of `wght`=0 (default) will be used. If flag bit 14 is clear, `wght`=.5 (from current glyph) will be used.

**Note:** While it is the (undocumented?) behavior of the `glyf` table that glyphs loaded are shifted to align their LSB to that specified in the `hmtx` table, much like regular Composite glyphs, this does not apply to component glyphs being loaded as part of a VariableComposite glyph.

**Note:** A static (non-variable) font that uses VarComposite glyphs, _would not_ have `fvar` / `avar` tables but _would_ have the `gvar` table. This combination is possible because the `gvar` table encodes its own number-of-axes, and the axes in this proposal are specified in the normalized space.
