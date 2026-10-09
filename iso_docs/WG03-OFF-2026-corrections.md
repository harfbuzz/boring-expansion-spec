# Proposed corrections and clarifications to Open Font Format fifth edition

**Committee:** ISO/IEC JTC 1/SC 29/WG 3

**Reference standard:** ISO/IEC 14496-22:2026, fifth edition, Open font format

**Status:** Draft contribution

**Date:** 9 October 2026

## Introduction

This contribution proposes six corrections and clarifications to the fifth
edition of Open Font Format (OFF). They address character-map fallback, an
ambiguous flag description, an omitted array count, an incorrect offset base,
component glyph origins, and handling of malformed variation regions.

Each proposal can be considered independently. Clause references and page
numbers refer to the published fifth edition; page numbers are the printed
numbers, rather than PDF viewer page numbers. Only material under **Proposed
replacement** is proposed for incorporation into the standard. The nature,
rationale, and compatibility discussions are explanatory material for the
working group.

| Proposal | Subject | Affected clauses | Nature |
| --- | --- | --- | --- |
| 1 | DMAP fallback and default Unicode variation sequences | 5.6.15 | Lookup clarification |
| 2 | Long-offset flag in gvar and GVAR | 7.3.4.1.1, 7.3.9.1.1 | Editorial correction |
| 3 | Missing count in VARC ConditionList | 7.3.10.3 | Omitted-field correction |
| 4 | SparseVariationRegion offset base and designation | 7.2.3.5 | Technical and terminology correction |
| 5 | Origin alignment of VARC component glyphs | 7.3.10.5 | Technical processing change |
| 6 | Processing of malformed variation regions | 7.1.7, 7.2.3.3, 7.2.3.5 | Relaxation of malformed-data processing requirements |

## 1. Clarify DMAP fallback and default Unicode variation sequence matches

### Nature and affected text

Clarification of the lookup rules in **5.6.15, DMAP—Delta map table**, printed
page **341**, without changing the table layout.

The first paragraph requires a lookup in `cmap` when a match is not found in
`DMAP`, but does not expressly address a mapping to glyph ID 0. The final
paragraph also describes DMAP subtables as superseding corresponding cmap
subtables in their entirety. That wording needs to be reconciled with the
fallback rule and the handling of default Unicode variation sequences (UVSes).

Source: [pull request #179](https://github.com/harfbuzz/boring-expansion-spec/pull/179).

### Rationale

DMAP allows a font in a collection to override selected entries in a shared
character map. In `cmap`, a mapping to glyph ID 0 already means that the
character is not covered. **5.1.2.1, Table overview**, explicitly states:

> Regardless of the encoding scheme, character codes that do not correspond
> to any glyph in the font should be mapped to glyph index 0. The glyph at
> this location shall be a special glyph representing a missing character,
> commonly known as .notdef.

DMAP uses the same subtable formats as cmap, so interpreting a mapping to
glyph ID 0 as a miss follows this existing meaning. It needs to have the same
fallback behavior as an absent mapping; otherwise it can conceal a usable
mapping in `cmap`. This applies both to ordinary character mappings and to
non-default UVS mappings in formats 14 and 15.

A match in a default UVS range is different: it successfully selects ordinary
character mapping for the base character. That ordinary lookup consults DMAP
before cmap. Falling through to the cmap UVS subtable after a successful DMAP
default-range match could instead select a non-default glyph, defeating the
DMAP result. These rules document the HarfBuzz and FontTools behavior described
in [pull request #179](https://github.com/harfbuzz/boring-expansion-spec/pull/179).

### Proposed replacement

In **5.6.15**, replace the first paragraph with:

> An optional DMAP table, if present, shall be consulted before the 'cmap'
> table for character-to-glyph lookups, including Unicode variation sequence
> lookups. If a DMAP lookup has no matching mapping or maps to glyph ID 0,
> the corresponding 'cmap' lookup shall be performed. This fallback rule
> applies to ordinary character mappings and non-default Unicode variation
> sequence mappings in formats 14 and 15. The DMAP table is identical in
> structure to the 'cmap' table.

Replace the final paragraph, beginning “Any subtables in a DMAP table”, with:

> DMAP mappings shall take priority over corresponding 'cmap' mappings,
> subject to the fallback rule above. A match in a default Unicode variation
> sequence range in a format 14 or 15 DMAP subtable is a successful match.
> The glyph shall be obtained by ordinary character mapping of the base
> character, consulting DMAP before 'cmap' and applying the ordinary-character
> fallback rule above. The 'cmap' Unicode variation sequence mapping shall not
> be consulted for a sequence that matches a DMAP default range.

### Compatibility

The binary format is unchanged. A DMAP mapping to glyph ID 0 cannot be used to
suppress an otherwise available cmap mapping. A default UVS range match in
DMAP takes precedence over any cmap UVS mapping for that sequence.

## 2. Identify the long-offset flag as bit 0 in gvar and GVAR

### Nature and affected text

Editorial correction of the `flags` field description in both
**7.3.4.1.1, 'gvar' header**, printed page **797**, and
**7.3.9.1.1, GVAR header**, printed page **834**.

Both descriptions refer to “bit 1” when selecting between 16-bit and 32-bit
offsets. The intended mask is `0x0001`.

Source: [issue #181](https://github.com/harfbuzz/boring-expansion-spec/issues/181).

### Rationale

Other flag definitions in OFF use zero-based numbering. For example, the
`head` flags in 5.1.3 begin at bit 0, and the simple-glyph flags in both
5.2.4.1.1 (`glyf`) and 5.2.8.1.1 (`GLYF`) identify mask `0x01` as bit 0.
Using “bit 1” for mask `0x0001` in the variation headers creates an ambiguity
that can cause an implementation to select mask `0x0002` instead.

The established encoding selects long offsets with the least significant bit,
mask `0x0001`. The OpenType gvar header also explicitly identifies that flag as
bit 0. The correction makes OFF internally consistent and preserves that
encoding. See the [OpenType gvar header definition](https://learn.microsoft.com/en-us/typography/opentype/spec/gvar#gvar-header)
and the [discussion in issue #181](https://github.com/harfbuzz/boring-expansion-spec/issues/181#issuecomment-5981166021).

### Proposed replacement

In **both 7.3.4.1.1 and 7.3.9.1.1**, replace the description of the
`uint16 flags` field with:

> Bit-field that gives the format of the offset array that follows. If bit 0
> (mask 0x0001) is clear, the offsets are uint16; if bit 0 (mask 0x0001) is
> set, the offsets are uint32.

### Compatibility

This is a wording correction. The long-offset flag remains mask `0x0001`.
Short offsets remain 16-bit stored values multiplied by two to obtain byte
offsets; long offsets remain unscaled 32-bit byte offsets.

## 3. Restore the omitted count in the VARC ConditionList table

### Nature and affected text

Correction of an omitted field in the **ConditionList table** definition in
**7.3.10.3, Variable Composite Description**, printed page **849**.

The published definition contains only `Offset32 conditionOffsets[]`, with no
field specifying the number of offsets. The correction restores the leading
32-bit count and specifies the array length. The replacement also corrects the
condition-format cross-reference from 6.2.9 to **6.2.10, Conditions and
Condition Sets**.

Source: [issue #176](https://github.com/harfbuzz/boring-expansion-spec/issues/176).

### Rationale

An offset array needs an explicit length. Neither the VARC header nor the
length of the enclosing VARC table determines the number of entries in this
subtable.

The original VARC definition uses `Array32Of<Offset32To<Condition>>`: a uint32
count followed by that many Offset32 entries. The count was omitted during
translation into the ISO proposal. HarfBuzz and FontTools already encode and
read the leading 32-bit count. See the
[original ConditionList definition](https://github.com/harfbuzz/boring-expansion-spec/blob/9c795dc2f8efe0b488a5dddfebf909594a23a280/VARC.md#L321-L326),
the [HarfBuzz definition](https://github.com/harfbuzz/harfbuzz/blob/9af58580c6ec7bc64695662d71b8d03934b3ac13/src/hb-ot-layout-common.hh),
and the [FontTools definition](https://github.com/fonttools/fonttools/blob/71b02bb93a61731f802f032f19aa1f9f634a2911/Lib/fontTools/ttLib/tables/otData.py).

### Proposed replacement

In **7.3.10.3**, replace the ConditionList definition with:

**ConditionList table**

| Type | Name | Description |
| --- | --- | --- |
| uint32 | conditionCount | The number of condition offsets in the conditionOffsets array. |
| Offset32 | conditionOffsets[conditionCount] | Array of offsets to condition tables (see 6.2.10), measured from the start of the ConditionList table. |

### Compatibility

This restores the original binary layout used by implementations. The first
offset follows the four-byte count, and offsets are measured from the start
of the ConditionList table, including that count. Existing fonts using this
layout require no conversion.

## 4. Correct the SparseVariationRegion offset base and table designation

### Nature and affected text

Technical correction of an offset base, together with a terminology correction,
in **7.2.3.5, MultiItem Variation Store**, printed page **769**.

The `sparseRegionAxisCoordinates[regionAxisCount]` offsets in each
SparseVariationRegion are currently described as relative to the enclosing
SparseVariationRegionList. They should be relative to the SparseVariationRegion
itself. The structure is also incorrectly designated a record.

Source: [issue #177](https://github.com/harfbuzz/boring-expansion-spec/issues/177).

### Rationale

A SparseVariationRegion is reached through an offset in the
`variationRegionOffsets` array. It is therefore a table under the table/record
distinction in **4.3, Data types**. Making its axis offsets relative to its own
start allows it to be read, written, or relocated without carrying the address
of an ancestor table.

This follows the original VARC structure and the table-relative offset handling
used by HarfBuzz and FontTools. See the
[original sparse-region definition](https://github.com/harfbuzz/boring-expansion-spec/blob/9c795dc2f8efe0b488a5dddfebf909594a23a280/VARC.md),
the [HarfBuzz sparse-region implementation](https://github.com/harfbuzz/harfbuzz/blob/9af58580c6ec7bc64695662d71b8d03934b3ac13/src/hb-ot-layout-common.hh),
and the [FontTools sparse-region definition](https://github.com/fonttools/fonttools/blob/71b02bb93a61731f802f032f19aa1f9f634a2911/Lib/fontTools/ttLib/tables/otData.py).

### Proposed replacement

In **7.2.3.5**, replace the description of
`variationRegionOffsets[regionCount]` in SparseVariationRegionList with:

> Array of offsets to SparseVariationRegion tables, measured from the start
> of the SparseVariationRegionList.

Replace the heading “SparseVariationRegion record” and its definition with:

**SparseVariationRegion table**

| Type | Name | Description |
| --- | --- | --- |
| uint16 | regionAxisCount | The number of axes for the region. |
| Offset32 | sparseRegionAxisCoordinates[regionAxisCount] | Array of offsets to SparseRegionAxisCoordinates records, measured from the start of the SparseVariationRegion table. |

### Compatibility

The field widths, counts, and order are unchanged. The `variationRegionOffsets`
in SparseVariationRegionList retain their existing list-relative base.

The corrected axis-offset base agrees with implementations using table-relative
offsets. A font encoded using the published list-relative axis-offset wording
requires those axis offsets to be recalculated. If the list starts at address
`L`, a region at `R`, and an axis structure at `A`, the corrected offsets are
`R - L` in `variationRegionOffsets` and `A - R` in the region's
`sparseRegionAxisCoordinates` array.

## 5. Use ordinary glyph-origin alignment for VARC component glyphs

### Nature and affected text

Technical change to the processing rule expressed in **NOTE 2** of
**7.3.10.5, Processing of Variable Composite Glyphs**, printed page **852**.

The note currently exempts VARC component glyphs from customary
left-side-bearing alignment when loading TrueType outlines. The proposal
removes that exception and specifies ordinary glyph-origin behavior for
components loaded from `glyf` or `GLYF`. The processing requirement is placed
in the main text, with an explanatory note following it.

Source: [issue #173](https://github.com/harfbuzz/boring-expansion-spec/issues/173).

### Rationale

VARC was originally developed as part of the glyf composite design, which
explains its original component-origin convention. As a separate table, VARC
implementations load leaf glyphs through ordinary glyph-loading paths.
The checks reported in issue #173 found that HarfBuzz, FontTools, and FreeType
apply ordinary left-side-bearing alignment for VARC glyf leaves. Preserving
this behavior avoids changing their rendered outlines and introducing special
handling around phantom points, hinting, and metrics.

The difference can be demonstrated in a static font. Let a leaf outline span
`x = 100..200`, with a stored horizontal left side bearing of 50, and let the
VARC component translate it by +30. Under the published exception, the result
spans `130..230`. Ordinary glyph loading first shifts the outline by
`50 - 100 = -50`, so applying the VARC translation produces `80..180`, as
reported for all three implementations. When the left side bearing equals
xMin, the two conventions agree and the discrepancy is concealed. See the
[example and implementation checks in issue #173](https://github.com/harfbuzz/boring-expansion-spec/issues/173).

### Proposed replacement

In **7.3.10.5**, replace NOTE 2 with the following normative paragraph and
explanatory note, leaving NOTE 3 in place:

> When a component outline is loaded from 'glyf' or GLYF, its glyph-origin
> convention shall be the same as for ordinary top-level loading of that glyph
> at the component's variation coordinates, including the usual
> left-side-bearing and phantom-point alignment. The Variable Component
> transformation shall be applied to the loaded outline after that alignment.
>
> NOTE 2 This origin convention applies when a glyph is loaded as a VARC
> component. It does not alter the placement of components within an ordinary
> 'glyf' or GLYF composite glyph.

### Compatibility

This changes the published processing rule to match the behavior reported for
HarfBuzz, FontTools, and FreeType. A font authored for the raw-outline-origin
exception can render differently under the corrected rule, particularly when
a leaf glyph's left side bearing differs from xMin. The proposal does not
introduce a new origin convention for CFF2 or change the hinting provisions
in 7.3.10.4.

## 6. Leave malformed variation-region processing to implementations

### Nature and affected text

Relaxation of prescribed processing for malformed per-axis variation-region
definitions in **7.1.7, Algorithm for interpolation of instance values**,
printed pages **739–742**. The clarification also applies to the region
formats in **7.2.2, Tuple variation store**, **7.2.3.3, Variation regions**,
and **7.2.3.5, MultiItem Variation Store**.

The published pseudocode assigns a per-axis scalar of 1 when start, peak, and
end coordinates are incorrectly ordered, or when a region crosses zero with
a nonzero peak. The proposal retains the validity requirements while leaving
the processing result for such malformed definitions unspecified. A zero peak
continues to give a per-axis scalar of 1.

Source: [issue #178](https://github.com/harfbuzz/boring-expansion-spec/issues/178),
including the [final proposal in the discussion](https://github.com/harfbuzz/boring-expansion-spec/issues/178#issuecomment-5963753485).

### Rationale

For a well-formed region, an instance coordinate of zero and a nonzero peak
give a scalar of zero. An implementation can therefore return zero after
examining only the peak and instance coordinates. The published ordering of
checks requires it to examine start and end coordinates first, because an
invalid definition changes the prescribed result to 1.

For example, `start = 0.5`, `peak = 0.25`, `end = 1.0`, and an instance
coordinate of zero form a malformed per-axis definition because start exceeds
peak. The published algorithm returns 1; the early zero-coordinate check
returns 0. Both approaches give identical results for well-formed regions.

The issue also affects intermediate tuples that reuse a cached zero scalar
from a shared peak-only tuple. Requiring a particular recovery result for
malformed start/end data can force implementations to revisit that cached
result. The discussion reports different behavior across HarfBuzz, FreeType,
Fontations, and FontTools and proposes retaining the validity rules without
prescribing recovery behavior. This permits existing fast paths and caching
without changing interpolation results for valid regions. See the
[examples and caching discussion in issue #178](https://github.com/harfbuzz/boring-expansion-spec/issues/178).

### Proposed replacement

**a. Region validity and malformed-data handling in 7.1.7**

Replace the paragraph beginning “In order for the definition of a region
within variation data to be valid” with:

> In order for a region definition within variation data to be valid, the
> start, peak, and end coordinate values shall be well ordered for each axis:
> the start coordinate shall be less than or equal to the peak coordinate,
> and the peak coordinate shall be less than or equal to the end coordinate.
> For an axis with a nonzero peak coordinate, the start and end coordinates
> shall not have opposite signs. A start coordinate below zero and an end
> coordinate above zero are permitted when the peak coordinate is zero.
>
> For an axis with a nonzero peak coordinate, this document does not specify
> the processing result if the per-axis region definition violates the
> coordinate-ordering requirements or the restriction on crossing zero. These
> validity and processing provisions apply to tuple variation stores, regular
> ItemVariationStore regions, and sparse MultiItemVariationStore regions.
> For an axis with a zero peak coordinate, the per-axis scalar shall be 1,
> regardless of the start and end coordinates.

**b. Interpolation pseudocode in 7.1.7**

Replace the sentence introducing the pseudocode with:

> The following pseudo-code specifies the interpolation algorithm for valid
> region definitions. It also specifies the per-axis scalar of 1 for an axis
> with a zero peak coordinate. It does not prescribe processing of invalid
> per-axis region definitions with a nonzero peak coordinate.

Within the per-axis loop, replace the initial validity-check comment, the two
branches assigning `AS = 1` for invalid definitions, the comment beginning
“Note: for remaining cases”, and the zero-peak branch with:

```c
/* If the peak is zero for some axis, then ignore the axis. */
if (peakCoords[i] == 0)
    AS = 1;

/* For a nonzero peak, this algorithm assumes a valid per-axis
   region definition. Processing of an invalid definition is
   not specified by this document. */
```

The existing `else if` for an instance coordinate outside the start/end range,
and the subsequent interpolation and accumulation steps, follow this block
unchanged.

**c. Cross-reference from regular variation regions**

In **7.2.3.3**, after the paragraph defining the range and ordering requirements
for `startCoord`, `peakCoord`, and `endCoord`, insert:

> Processing of malformed per-axis region definitions is governed by 7.1.7.
> A zero peak coordinate gives a per-axis scalar of 1.

**d. Cross-reference from sparse variation regions**

In **7.2.3.5**, immediately after the SparseRegionAxisCoordinates definition,
insert:

> The coordinate-ordering requirements, zero-crossing restriction, and
> malformed-data processing provisions in 7.1.7 apply to the startCoordinate,
> peakCoordinate, and endCoordinate fields. A zero peakCoordinate gives a
> per-axis scalar of 1.

### Compatibility

Well-formed regions retain their existing interpolation results in all three
store formats. Zero-peak axes, including axes used for constant contributions,
retain scalar 1.

For malformed definitions with a nonzero peak, implementations may continue
to use the existing axis-ignore behavior, an early zero-coordinate return,
or cached zero scalars without a prescribed recovery result. The relaxation
applies at all instance coordinates; it follows the final discussion proposal
to leave malformed-data processing unspecified, rather than mandating a
particular optimization. Such data remain invalid and font producers cannot
rely on interoperable rendering results for them.
