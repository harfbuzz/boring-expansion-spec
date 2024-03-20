# `VARC` table: Variable Composites / Components

This table encodes *variable-composite* glyphs in a way that can be combined
with any of `glyf`/`gvar` or `CFF2`. Since the primary purpose of
variable-composites is to make font files (especially the CJK fonts) smaller,
several new data-structures are introduced to achieve more compression compared
to an equivalent non-variable-composite font.


## Data Structures

The following foundational data-structures are used in this able:

- `uint32var` is a variable-length encoding of `uint32` values, which uses 1 to
  5 bytes depending on the magnitude of the value being encoded. It borrows
  ideas from the UTF-8 encoding, but is more efficient because it does not have
  some of the random-access properties of UTF-8 encoding.

- `TupleValues` is borrowed from the `TupleVariationStore` [Packed
  Deltas](https://learn.microsoft.com/en-us/typography/opentype/spec/otvarcommonformats#packed-deltas)
  with minor modification to allow storing 32-bit values.

- `CFF2IndexOf` is simply a CFF2-style `Index` structure containing data of a
  particular type.  CFF2 `Index` is defined
  [here](https://learn.microsoft.com/en-us/typography/opentype/spec/cff2#5-index-data).
  The purpose of `CFF2IndexOf` is to efficiently store a list of variable-sized
  data, for example glyph records.

- `MultiItemVariationStore`: This is a new data-structure. It is a hybrid
  between `ItemVariationStore`, and `TupleVariationStore`, borrowing ideas from
  both and improving upon them for more efficient storage of variations of
  *tuples* of numbers.


### `uint32var`

To be added to [The OpenType Font File Data
Types](https://learn.microsoft.com/en-us/typography/opentype/spec/otff#data-types)

A `uint32var` is a variable-length encoding of a `uint32`. If the number fits in
7 bits, then it is encoded as is. Otherwise, it encodes the number of
subsequent bytes as top bits of the first byte. The subsequent bytes use the
full 8 bits for storage.

*TODO:* Add table of encoding bytes.

Here is Python code to read and write `uint32var` values:

```python
def read_uint32var(data, i):
    """Read a variable-length uint32 number from data starting at index i.

    Return the number and the next index.
    """

    b0 = data[i]
    if b0 < 0x80:
        return b0, i + 1
    elif b0 < 0xC0:
        return (b0 - 0x80) << 8 | data[i + 1], i + 2
    elif b0 < 0xE0:
        return (b0 - 0xC0) << 16 | data[i + 1] << 8 | data[i + 2], i + 3
    elif b0 < 0xF0:
        return (b0 - 0xE0) << 24 | data[i + 1] << 16 | data[i + 2] << 8 | data[
            i + 3
        ], i + 4
    else:
        return (b0 - 0xF0) << 32 | data[i + 1] << 24 | data[i + 2] << 16 | data[
            i + 3
        ] << 8 | data[i + 4], i + 5
```
```python
def write_uint32var(v):
    """Write a variable-length uint32 number.

    Return the data.
    """
    if v < 0x80:
        return struct.pack(">B", v)
    elif v < 0x4000:
        return struct.pack(">H", (v | 0x8000))
    elif v < 0x200000:
        return struct.pack(">L", (v | 0xC00000))[1:]
    elif v < 0x10000000:
        return struct.pack(">L", (v | 0xE0000000))
    else:
        return struct.pack(">B", 0xF0) + struct.pack(">L", v)
```

### `TupleValues`

`TupleValues` is similar to the `TupleVariationStore` [Packed
Deltas](https://learn.microsoft.com/en-us/typography/opentype/spec/otvarcommonformats#packed-deltas)
with a minor modification: if the top two bits of the control byte
(`DELTAS_ARE_ZERO` and `DELTAS_ARE_WORDS`) are both set, then the following
values are 32-bit. That difference should be incorporated in the Packed Deltas
section of `TupleVariationStore`, and is backwards-compatible because the two
top bits can currently never be set at the same time.

`TupleValues` can be used in two different ways:
- When the number of values to be decoded is known in advance, decoding stops
  when the needed number of values are decoded.
- When the number of values to be decoded is _not_ known in advance, but the
  number of encoded bytes is known, then values are decoded until the bytes are
  depleted.


### `CFF2IndexOf`

`CFF2IndexOf` is simply a CFF2-style `Index` structure containing data of a particular type.
CFF2 `Index` is defined
[here](https://learn.microsoft.com/en-us/typography/opentype/spec/cff2#5-index-data).
The purpose of `CFF2IndexOf` is to efficiently store a list of variable-sized
data, for example glyph records, or other data.

Note that the `count` field of CFF2 `Index` structure is `uint32`, unlike the
count field of CFF `Index` structure.

### `MultiItemVariationStore`

To be added to [OpenType Font Variations Common Table
Formats](https://learn.microsoft.com/en-us/typography/opentype/spec/otvarcommonformats).

A `MultiItemVariationStore` is a new data-structure. It is a hybrid between
`ItemVariationStore`, and `TupleVariationStore`, borrowing ideas from both and
improving upon them for more efficient storage of variations of *tuples* of
numbers.

Like `ItemVariationStore`, entries are addressed using a 32-bit `VarIdx`,
with the top 16 bits called "outer" index, and lower 16 bits called the "inner"
index.

Whereas the `ItemVariationStore` stores deltas for a single scalar value for
each `VarIdx`, the `MultiItemVariationStore` stores deltas for a tuple for each
`VarIdx`. Compared to `ItemVariationStore`, the `MultiItemVariationStore` uses a
sparse encoding of the _active_ axes for each region, which is more efficient
in fonts with high number of axes.

Compared to `TupleVariationStore`, the `MultiItemVariationStore` is optimized
for smaller tuples and allows tuple-sharing, which is important for its
efficiency over the `TupleVariationStore`. It also does not have some of the
limitations that `TupleVariationStore` has, like the total size of an entry
being limited to 64kb.

The following structures form the `MultiItemVariationStore`.  Its processing is
fairly similar to that of the `ItemVariationStore`, except that the deltas
encoded for each entry consist of multiple numbers per region. The
`TupleValues` for each entry is as such the concatenation of the tuple deltas
for each region.

```c++
struct MultiItemVariationStore
{
  uint16 format; // Set to 1
  LOffsetTo<SparseVariationRegionList> variationRegionListOffset;
  uint16 itemVariationDataCount;
  Offset32To<MultiItemVariationData> itemVariationDataOffsets[itemVariationDataCount];
};
```
```c++
struct SparseVariationRegionList
{
  uint16 regionCount;
  Offset32To<SparseVariationRegion> variationRegionOffsets[regionCount];
}
```
```c++
struct SparseVariationRegion
{
  uint16 regionAxisCount;
  SparseRegionAxisCoordinates regionAxes[regionAxisCount];
};
```
```c++
struct SparseRegionAxisCoordinates
{
  uint16 axisIndex;
  F2DOT14 startCoord;
  F2DOT14 peakCoord;
  F2DOT14 endCoord;
};
```

```c++
struct MultiItemVariationData
{
  uint8 Format; // 1
  uint16 regionIndexCount;
  uint16 regionIndexes[regionIndexCount];
  CFF2IndexOf<TupleValues> deltaSets;
};
```

The `deltaSets` in a `MultiItemVariationData` table store the delta-set for a
single tuple, addressed by the "inner" index of the `VarIdx`, whereas a
`MultiItemVariationData` table itself represents the data for all values
sharing the same "outer" index.

The `TupleValues` for each entry are the concatenation of the tuple deltas for
each region. The length of the tuple is calculate by dividing the number of
entries in the `TupleValues` structure by the number of regions.


## Variable Composite Description

A Variable Composite record is a concatenation of Variable Component records.
Variable Component records have varying sizes.

```c++
struct VarCompositeGlyph
{
  VarComponent components[];
};
```

When decoding a `VarCompositeGlyph`, the decoder stops when the bytes for the
glyph are depleted.

## Variable Component Record

A Variable Component record encodes one component's glyph index, variations
location, and transformation in a variable-sized and efficient manner.

| type | name | notes |
|-|-|-|
| uint32var | `flags` | See below. |
| GlyphID16 or GlyphID24 | `gid` | This is a GlyphID16 if `GID_IS_24BIT` bit of `flags` is clear, else GlyphID24. |
| uint32var | `axisIndicesIndex` | Optional, only present if `HAVE_AXES` bit of `flags` is set. |
| TupleValues | `axisValues` | Optional, only present if `HAVE_AXES` bit of `flags` is set. The axis value for each axis. |
| uint32var | `axisValuesVarIndex` | Optional, only present if `AXIS_VALUES_HAVE_VARIATION` bit of `flags` is set. |
| uint32var | `transformVarIndex` | Optional, only present if `TRANSFORM_HAS_VARIATION` bit of `flags` is set. |
| FWORD | `TranslateX` | Optional, only present if `HAVE_TRANSLATE_X` bit of `flags` is set. |
| FWORD | ` TranslateY` | Optional, only present if `HAVE_TRANSLATE_Y` bit of `flags` is set. |
| F4DOT12 | `Rotation` | Optional, only present if `HAVE_ROTATION` bit of `flags` is set. Counter-clockwise. |
| F6DOT10 | `ScaleX` | Optional, only present if `HAVE_SCALE_X` bit of `flags` is set. |
| F6DOT10 | `ScaleY` | Optional, only present if `HAVE_SCALE_Y` bit of `flags` is set. |
| F4DOT12 | `SkewX` | Optional, only present if `HAVE_SKEW_X` bit of `flags` is set. Counter-clockwise. |
| F4DOT12 | `SkewY` | Optional, only present if `HAVE_SKEW_Y` bit of `flags` is set. Counter-clockwise. |
| FWORD | `TCenterX` | Optional, only present if `HAVE_TCENTER_X` bit of `flags` is set. |
| FWORD | ` TCenterY` | Optional, only present if `HAVE_TCENTER_Y` bit of `flags` is set. |
| uint32var[] | `reserved` | Optional, process and discard one `uint32var` per each set bit in `RESERVED`. |

### Variable Component Flags

| bit number | name |
|-|-|
| 0 | `RESET_UNSPECIFIED_AXES` |
| 1 | `HAVE_AXES` |
| 2 | `AXIS_VALUES_HAVE_VARIATION` |
| 3 | `TRANSFORM_HAS_VARIATION` |
| 4 | `HAVE_TRANSLATE_X` |
| 5 | `HAVE_TRANSLATE_Y` |
| 6 | `HAVE_ROTATION` |
| 7 | `USE_MY_METRICS` |
| 8 | `HAVE_SCALE_X` |
| 9 | `HAVE_SCALE_Y` |
| 10 | `HAVE_TCENTER_X` |
| 11 | `HAVE_TCENTER_Y` |
| 12 | `GID_IS_24BIT` |
| 13 | `HAVE_SKEW_X` |
| 14 | `HAVE_SKEW_Y` |
| 15-31 | `RESERVED`. Set to 0 |

The flags are arranged in the order of likely use, to minimize the storage
bytes of the `flags` field.

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

The rotation and skew parameters are in angles as multiples of Pi.

Two new types, `F4DOT12` and `F6DOT10` need to be added to the [Data
Types](https://learn.microsoft.com/en-us/typography/opentype/spec/otff#data-types)
section of the specification.

Details of how to build a transformation matrix, as Python code:
```python
# Using fontTools.misc.transform.Transform
t = Transform()  # Identity
t = t.translate(TranslateX + TCenterX, TranslateY + TCenterY)
t = t.rotate(Rotation * math.pi)
t = t.scale(ScaleX, ScaleY)
t = t.skew(-SkewX * math.pi, SkewY * math.pi)
t = t.translate(-TCenterX, -TCenterY)
```

## `VARC` table header

The top-level `VARC` table header is as follows:
```c++
struct VARC
{
  uint16 majorVersion; // 1
  uint16 minorVersion; // 0
  Offset32To<Coverage> coverage;
  Offset32To<MultiItemVariationStore> varStore;
  Offset32To<CFF2IndexOf<TupleValues>> axisIndicesList;
  Offset32To<CFF2IndexOf<VarCompositeGlyph>> glyphRecords;
};
```

The `coverage` table enumerates all glyphs that have Variable-Composite records
in this table. The records are encoded in `glyphRecords` by coverage index as
usual.

The `varStore` stores the variations of axis-values and transform values as
referenced in the Variable Component records.

The `axisIndicesList` encodes a list of axis-indices tuples, and is shared by
the glyphs which address it using an index.


## Processing

The component glyphs to be loaded use the coordinate values specified (with any
variations applied if present).

For any unspecified axis, the value used depends on flag
`RESET_UNSPECIFIED_AXES`. If the flag is set, then the normalized value zero is
used. If the flag is clear the axis values from current glyph being processed
(which itself might recursively come from the font or its own parent glyphs)
are used.  For example, if the font variations have `wght`=.25 (normalized),
and current glyph being processed is using `wght`=.5 because it was referenced
from another VarComposite glyph itself, when referring to a component that does
_not_ specify the `wght` axis, if the flag bit is set, then the value of
`wght`=0 (default) will be used. If the flag bit is clear, `wght`=.5 (from
current glyph) will be used.

The component location and transform can vary. These variations are stored in
the `MultiItemVariationStore` data-structure. The variations for location are
referred to by the `axisValuesVarIndex` member of a component if any, and
variations for transform are referred to by the `transformVarIndex` if any. For
transform variations, only those fields specified and as such encoded as per
the flags have variations.

The component then is recursively loaded from the `VARC` table, if it is
present in this table; otherwise, falls back to `glyf`/`gvar`, `CFF2`, or any
other mechanism to get raw outlines.  An exception to the recursive rule is
that if a component refers to the glyphID of the current glyph, instead of
recursing (which would result in infinite recursion), the glyph outline for the
component is loaded directly instead, as if the glyphID had no `VARC` entry.

**Note:** While it is the (undocumented?) behavior of the `glyf` table that
glyphs loaded are shifted to align their LSB to that specified in the `hmtx`
table, much like regular Composite glyphs, this does not apply to component
glyphs being loaded as part of a variable-composite glyph.

**Note:** A static (non-variable) font that uses the `VARC` table, _would not_
have `fvar` / `avar` tables but _would_ have the `gvar` table in a
TrueType-flavored font, or `CFF2` variations in a CFF-flavored font. This is
because the components themselves store their variables in the classic way. The
exception to this situation is a font with `VARC` table that does NOT vary the
component locations and transforms, and does not encode any location for the
components either. In practice, such a font would just use the `VARC` table
in the manner of classic components in a `glyf` table.
