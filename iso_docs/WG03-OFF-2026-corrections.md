# Proposed corrections and clarifications to Open Font Format fifth edition

**Committee:** ISO/IEC JTC 1/SC 29/WG 3

**Reference standard:** ISO/IEC 14496-22:2026, fifth edition, Open font format

**Status:** Draft contribution

**Date:** 10 October 2026

## Introduction

The primary purpose of the proposed changes and corrections is to provide
additional information and clarify technical details for implementers of Open
Font Format (OFF). The changes address DMAP lookup rules, flag numbering in
the glyph variation headers, the ConditionList array count, sparse-region
offset bases, component glyph origins, and processing of malformed variation
regions.

The DMAP clarification follows the existing meaning of glyph index 0 in the
'cmap' table overview (5.1.2.1): the character is not covered, and glyph index 0
represents the missing character, .notdef. Restoring the ConditionList count is
necessary for safe parsing and use of a VARC table containing that structure;
without it, readers cannot determine the array bounds or validate condition
indices.

All subclause references below refer to ISO/IEC 14496-22:2026.

## 1. DMAP lookup rules

In subclause **5.6.15, DMAP—Delta map table**, replace the first paragraph
with:

> An optional DMAP table, if present, shall be consulted before the 'cmap'
> table for character-to-glyph lookups, including Unicode variation sequence
> lookups. If a DMAP lookup has no matching mapping or maps to glyph ID 0,
> the corresponding 'cmap' lookup shall be performed. This fallback rule
> applies to ordinary character mappings and non-default Unicode variation
> sequence mappings in formats 14 and 15. The DMAP table is identical in
> structure to the 'cmap' table.

In the same subclause, replace the final paragraph, beginning “Any subtables
in a DMAP table”, with:

> DMAP mappings shall take priority over corresponding 'cmap' mappings,
> subject to the fallback rule above. A match in a default Unicode variation
> sequence range in a format 14 or 15 DMAP subtable is a successful match.
> The glyph shall be obtained by ordinary character mapping of the base
> character, consulting DMAP before 'cmap' and applying the ordinary-character
> fallback rule above. The 'cmap' Unicode variation sequence mapping shall not
> be consulted for a sequence that matches a DMAP default range.

## 2. Long-offset flag in gvar and GVAR

In subclauses **7.3.4.1.1, 'gvar' header**, and **7.3.9.1.1, GVAR header**,
replace the description of the `uint16 flags` field in each header definition
with:

> Bit-field that gives the format of the offset array that follows. If bit 0
> (mask 0x0001) is clear, the offsets are uint16; if bit 0 (mask 0x0001) is
> set, the offsets are uint32.

## 3. ConditionList table

In subclause **7.3.10.3, Variable Composite Description**, replace the
ConditionList table definition with:

**ConditionList table**

| Type | Name | Description |
| --- | --- | --- |
| uint32 | conditionCount | The number of condition offsets in the conditionOffsets array. |
| Offset32 | conditionOffsets[conditionCount] | Array of offsets to condition tables (see 6.2.10), measured from the start of the ConditionList table. |

## 4. SparseVariationRegion offset base and designation

In subclause **7.2.3.5, MultiItem Variation Store**, replace the description
of `variationRegionOffsets[regionCount]` in SparseVariationRegionList with:

> Array of offsets to SparseVariationRegion tables, measured from the start
> of the SparseVariationRegionList.

In the same subclause, replace the heading “SparseVariationRegion record”
and its definition with:

**SparseVariationRegion table**

| Type | Name | Description |
| --- | --- | --- |
| uint16 | regionAxisCount | The number of axes for the region. |
| Offset32 | sparseRegionAxisCoordinates[regionAxisCount] | Array of offsets to SparseRegionAxisCoordinates records, measured from the start of the SparseVariationRegion table. |

## 5. VARC component glyph origins

In subclause **7.3.10.5, Processing of Variable Composite Glyphs**, replace
NOTE 2 with the following paragraph and note:

> When a component outline is loaded from 'glyf' or GLYF, its glyph-origin
> convention shall be the same as for ordinary top-level loading of that glyph
> at the component's variation coordinates, including the usual
> left-side-bearing and phantom-point alignment. The Variable Component
> transformation shall be applied to the loaded outline after that alignment.
>
> NOTE 2 This origin convention applies when a glyph is loaded as a VARC
> component. It does not alter the placement of components within an ordinary
> 'glyf' or GLYF composite glyph.

## 6. Malformed variation regions

In subclause **7.1.7, Algorithm for interpolation of instance values**,
replace the paragraph beginning “In order for the definition of a region within
variation data to be valid” with:

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

In the same subclause, replace the sentence “The following pseudo-code
provides a specification of the interpolation algorithm:” with:

> The following pseudo-code specifies the interpolation algorithm for valid
> region definitions. It also specifies the per-axis scalar of 1 for an axis
> with a zero peak coordinate. It does not prescribe processing of invalid
> per-axis region definitions with a nonzero peak coordinate.

In the per-axis loop of the pseudocode in the same subclause, replace the
block from the comment beginning “If a region definition is not valid in
relation to some axis” through the `AS = 1;` statement associated with
`else if (peakCoords[i] == 0)` with:

```c
/* If the peak is zero for some axis, then ignore the axis. */
if (peakCoords[i] == 0)
    AS = 1;

/* For a nonzero peak, this algorithm assumes a valid per-axis
   region definition. Processing of an invalid definition is
   not specified by this document. */
```

In subclause **7.2.3.3, Variation regions**, after the paragraph beginning
“The three values shall all be within the range -1.0 to +1.0”, insert the
following paragraph:

> Processing of malformed per-axis region definitions is governed by 7.1.7.
> A zero peak coordinate gives a per-axis scalar of 1.

In subclause **7.2.3.5, MultiItem Variation Store**, immediately after the
SparseRegionAxisCoordinates definition, insert the following paragraph:

> The coordinate-ordering requirements, zero-crossing restriction, and
> malformed-data processing provisions in 7.1.7 apply to the startCoordinate,
> peakCoordinate, and endCoordinate fields. A zero peakCoordinate gives a
> per-axis scalar of 1.
