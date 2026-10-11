# Proposed technical corrections and clarifications to ISO/IEC 14496-22:2026 (Open Font Format)

**Committee:** ISO/IEC JTC 1/SC 29/WG 3

**Reference standard:** ISO/IEC 14496-22:2026, fifth edition, Open font format

**Status:** Draft contribution

**Date:** 11 October 2026

## Introduction

The primary purpose of the proposed changes and corrections is to provide
additional information and clarify technical details for implementers of Open
Font Format (OFF). The changes address DMAP lookup rules, flag numbering in
the glyph variation headers, the ConditionList array count, sparse-region
offset bases, component glyph origins, processing of malformed variation
regions, and the distinction between default user-scale axis values and final
normalized coordinates.

The DMAP clarification follows the existing meaning of glyph index 0 in the
'cmap' table overview (5.1.2.1): the character is not covered, and glyph index 0
represents the missing character, .notdef. Restoring the ConditionList count is
necessary for safe parsing and use of a VARC table containing that structure;
without it, readers cannot determine the array bounds or validate condition
indices.

The normalization changes distinguish the unvaried location, at which all final
normalized coordinates are zero, from the default user-scale axis values, which
can be remapped to a different location by 'avar' version 2. They specify that
ordinary variation deltas do not apply at the unvaried location, while preserving
axis remapping and VARC processing at that location.

All subclause references below refer to ISO/IEC 14496-22:2026.

## 1. DMAP lookup rules

In subclauses **5.1.2.1, Table overview**, and **5.6.15, DMAP—Delta map
table**, replace the paragraph beginning “An optional DMAP table, if present”
with:

> An optional DMAP table, if present, shall be consulted before the 'cmap'
> table for character-to-glyph lookups, including Unicode variation sequence
> lookups. If a DMAP lookup finds no mapping, or an ordinary character
> mapping or non-default Unicode variation sequence mapping yields glyph
> index 0, the corresponding 'cmap' lookup shall be performed. This includes
> non-default Unicode variation sequence mappings in formats 14 and 15.
> A match in a default Unicode variation sequence range is handled as
> specified in 5.6.15. The DMAP table is identical in structure to the 'cmap'
> table.

In subclause **5.6.15**, replace the final paragraph, beginning “Any subtables
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

In the VarComponentRecord definition in the same subclause, replace the
description of `conditionIndex` with:

> Zero-based index into the conditionOffsets array in the ConditionList
> table. The index shall be less than conditionCount. Optional; present
> only if the HAVE_CONDITION bit of flags is set.

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
> 'glyf' or GLYF composite glyph. For a simple, unhinted glyph without variation
> adjustments, the horizontal alignment translates the outline by lsb − xMin,
> where lsb is obtained from 'hmtx' or HMTX and xMin from 'glyf' or GLYF,
> respectively.

## 6. Malformed per-axis region definitions

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
> validity and processing provisions apply to tuple variation stores,
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

In subclause **7.2.3.3, Variation regions**, replace the paragraph beginning
“The three values shall all be within the range -1.0 to +1.0” with:

> The three values shall all be within the range -1.0 to +1.0. The
> coordinate-ordering requirements, zero-crossing restriction, and processing
> of malformed per-axis region definitions are specified in 7.1.7. A zero
> peakCoord gives a per-axis scalar of 1, regardless of startCoord and
> endCoord.

In subclause **7.2.3.5, MultiItem Variation Store**, immediately after the
SparseRegionAxisCoordinates definition, insert the following paragraph:

> The coordinate-ordering requirements, zero-crossing restriction, and
> malformed-data processing provisions in 7.1.7 apply to the startCoordinate,
> peakCoordinate, and endCoordinate fields. A zero peakCoordinate gives a
> per-axis scalar of 1. Axes not represented in the sparse region do not
> affect its scalar. When regionAxisCount is zero, the region scalar is 1.

## 7. Default axis values and zero normalized coordinates

In subclause **5.1.3, head—Font header**, in the paragraph beginning
“Also, in a variable font, the minimum or maximum x or y values”, replace
“the default instance of that glyph” with “the unvaried outline of that
glyph”, and replace “for a non-default instance of the font” with “for a
variation instance of the font”.

In subclauses **5.1.5, hmtx—Horizontal metrics**, and **5.1.11,
HMTX—Horizontal metrics**, in the paragraph beginning “In a variable font
with TrueType outline data, the left side bearing value”, replace the
sentence “These values apply only to the default instance of the variable
font: non-default instances may have different side bearing values” with:

> These are unvaried side bearing values. Variation adjustments can be
> required for any instance, including the default user-scale instance,
> as specified in 7.1.7.

In those same paragraphs, replace “default-instance values” with “unvaried
values”. In the paragraph beginning “In a font with CFF version 1 outline
data” in each of those subclauses, replace “for non-default instances” with
“for variation instances”.

In subclauses **5.2.4.1, Table structure**, and **5.2.8.1, Table structure**,
in the paragraph beginning “In a variable font, the minimum and maximum
x or y values of control points can vary”, replace “the default instance
(that is, for the glyph description in this table)” with “the unvaried
glyph description in this table”, and replace “for a non-default instance
of a glyph” with “for a variation instance of a glyph”.

In subclause **6.2.2, OFF Layout and Font variations**, replace the sentence
“The font-unit values given in these tables apply to the default instance
of a variable font” with:

> The font-unit values given in these tables are unvaried values. Variation
> adjustments can be required for any instance, including the default
> user-scale instance, as specified in 7.1.7.

In subclause **6.2.10, Conditions and Condition Sets**, in the
ConditionFormat2 table, replace the description of the `int16 default`
field with:

> Unvaried value before application of variation deltas.

In subclauses **6.3.1.2, BASE table and OFF Font Variations**, and
**6.3.3.2, GPOS table and OFF Font Variations**, replace the sentence
“In a variable font, these X and Y values apply to the default instance
and may need to be adjusted for the current variation instance” with:

> In a variable font, these X and Y values are unvaried values and may
> need to be adjusted for the current variation instance, including the
> default user-scale instance, as specified in 7.1.7.

In subclause **6.3.1.2**, in the paragraph beginning “The VariationIndex
table is a variant of a Device table”, replace the sentence “This is done
so that the default instance of a variable font can be compatible with
applications that do not support Font Variations” with:

> This allows applications that do not support Font Variations to use the
> unvaried BASE values, which need not be the values obtained for the
> default user-scale instance.

In subclause **6.3.5.3, JSTF table and OFF font variations**, replace the
sentence “In a variable font, these X and Y values apply to the default
instance, and can require adjustment for different variation instances”
with:

> In a variable font, these X and Y values are unvaried values. Variation
> adjustments can be required for any instance, including the default
> user-scale instance, as specified in 7.1.7.

In subclause **7.1.1, General**, replace the paragraph beginning “A variable
font has a default instance, with axis parameter values set to the defaults”
with:

> A variable font has a default instance with user-scale coordinates set to
> the default values defined for each axis in the 'fvar' table. The font
> provides unvaried values for data items such as glyph outline points and
> metrics. Whether variation deltas are applied to those values depends on
> the final normalized coordinates after any 'avar' processing, as specified
> in 7.1.7. The default user-scale axis values can result in nonzero final
> normalized coordinates when an 'avar' version 2 table is present. The
> default instance can therefore require variation processing.

In subclause **7.1.2, Terminology**, replace NOTE 3 under the definition of
Design-variation space with:

> NOTE 3 A variation space can have one or more axes. In a variable font,
> the variation space is bounded by the minimum and maximum user-scale
> values specified in the 'fvar' table. The zero origin in user-scale
> coordinates has no special significance. The zero origin in final
> normalized coordinates identifies the unvaried location for glyph and
> metric data, subject to the VARC exception in 7.1.7. The default user-scale
> axis values do not necessarily select that location when 'avar' version 2
> is used.

In the same subclause, immediately before the definition of Variation data,
insert the following definition:

> Unvaried value: The value of a font data item before application of
> variation deltas. In descriptions of variation interpolation, a default
> value means this unvaried value, which need not be the value obtained for
> the default user-scale instance.

In the same subclause, replace NOTE 6 under the definition of Normalized
coordinate scale with:

> NOTE 6 The 'fvar' table specifies user-scale minimum, default, and maximum
> values for each axis. Initial normalization maps these values to -1, 0,
> and 1, respectively, except that an endpoint equal to the default value
> maps to 0. The 'avar' segment maps preserve -1, 0, and 1. Additional
> coordinate adjustments in 'avar' version 2 can change these mappings.
> Variation data use normalized coordinates; the coordinates used for each
> processing stage are specified in 7.1.4, 7.3.1.3, and 7.3.10.5.

In subclause **7.1.3, Variation space, default instances and adjustment
deltas**, replace the paragraph beginning “The variable font has a default
instance, which corresponds to the position in the variation space” with:

> The variable font has a default instance whose user-scale coordinates are
> the default values for each axis specified in the 'fvar' table. The font
> also contains unvaried values for data items, such as the grid coordinates
> of outline points in the 'glyf' table. These values are used without
> variation adjustments when the final normalized coordinates are zero for
> all axes, subject to the exceptions in 7.1.7. If 'avar' version 2 processing
> maps the default user-scale coordinates to nonzero final normalized
> coordinates, variation adjustments can apply to the default instance.

In the same subclause, replace the paragraph beginning “All other instances
have non-default coordinate values” with:

> Other instances select non-default user-scale values for one or more
> axes. Their final normalized coordinates determine the applicable
> adjustment deltas, as specified in 7.1.7. The examples and figures in this
> subclause and 7.1.5 that equate the default instance with unvaried data
> assume that normalization maps the default user-scale values to zero for
> all axes and that VARC processing does not alter the glyph outlines.

In the same subclause, replace the sentence “Also, a consideration in the
choice of default is desired behavior in legacy applications or platforms
that do not support Font Variations: in such software, only the default
instance of a variable font will be supported” with:

> Another consideration is the desired behavior in applications or
> platforms that do not support Font Variations: such software uses the
> unvaried font data, which need not represent the default user-scale
> instance when 'avar' version 2 is used.

In subclause **7.1.4, Coordinate scales and normalization**, replace the
paragraph beginning “Whereas the definitions in the 'fvar' table are
expressed in user coordinates” with:

> Whereas the definitions in the 'fvar' table are expressed in user
> coordinates, variation data use normalized coordinates. Initial
> normalization maps the minimum, default, and maximum user-scale values
> for each axis to -1, 0, and 1, respectively, except that an endpoint equal
> to the default value maps to 0. Any 'avar' processing is then applied to
> obtain the final normalized coordinates. The following figures illustrate
> the initial normalization before 'avar' processing.

In subclause **7.1.4, Coordinate scales and normalization**, replace the
paragraph beginning “The normalization transformation uses a default
transformation followed by a secondary modification” with:

> The normalization transformation first applies the default transformation
> and then applies any mappings defined in the 'avar' table. The segment maps
> preserve the normalized coordinates -1, 0, and 1; they affect only the
> mapping of intervening values.
> In 'avar' version 2, additional coordinate adjustments are then computed
> using an ItemVariationStore, as specified in 7.3.1.3. These adjustments can
> change the coordinates produced by the segment maps, including coordinates
> of -1, 0, or 1. In particular, default user-scale axis values can result in
> nonzero final normalized coordinates.

In the same subclause, after the sentence “See 7.3.1 for details on the
structures mentioned above”, insert the following paragraph:

> For an 'avar' version 2 table, the values produced by the segment-mapping
> algorithm above are intermediate normalized coordinates. The additional
> processing specified in 7.3.1.3 shall be performed to obtain the final
> normalized coordinates. This processing shall not be omitted solely because
> all intermediate normalized coordinates are zero.

In subclause **7.1.5, Variation data**, in the paragraph beginning “This
general case is not the most common in practice”, replace “the zero origin
— the default instance” with:

> the zero origin in normalized coordinates

In subclause **7.1.6, Variation data tables and miscellaneous requirements**,
in NOTE 1, replace the sentence beginning “In particular, the xMin, yMin,
xMax, yMax, macStyle and lowestRecPPEM fields” with:

> In particular, variation data are not supported for the xMin, yMin,
> xMax, yMax, macStyle, or lowestRecPPEM fields in the font header ('head')
> table. The bounding rectangle fields can fail to encompass outlines
> produced by variation processing, even for the default user-scale
> instance. The macStyle and lowestRecPPEM fields describe the default
> user-scale instance.

In subclause **7.1.7, Algorithm for interpolation of instance values**,
insert the following as the first two paragraphs:

> Except for the processing specified in the next paragraph, when the
> normalized coordinates used to evaluate variation data are zero for all
> axes, variation deltas shall not be applied to glyph outlines, metrics,
> or other font data. The unvaried values shall be used. For processing
> that uses the font instance coordinates, this zero-coordinate rule uses
> the final normalized coordinates after all applicable 'avar' processing,
> including the specified clamping and rounding. It applies even when variation data
> include regions with no active axes, such as regions whose peak
> coordinates are all zero. An implementation may omit evaluation of
> variation deltas in this case.
>
> The zero-coordinate rule does not apply to computation of axis-coordinate
> adjustments in 'avar' version 2 or to VARC processing, including evaluation
> of its conditions and MultiItemVariationStore. For ordinary outline and
> metric variation data used when loading a VARC component from 'glyf',
> GLYF, or CFF2, the rule uses the component's effective normalized
> coordinates after application of its axis values and axis-value deltas.
> A component with nonzero effective coordinates can therefore require
> ordinary variation processing even when all font instance coordinates are
> zero. The rule suppresses variation deltas; evaluation of condition tables
> and FeatureVariations tables remains governed by their respective
> subclauses.

In the same subclause, immediately before the sentence introducing the
interpolation pseudocode, insert the following paragraph:

> If the zero-coordinate rule applies, netAdjustment shall be zero and
> interpolatedValue shall equal defaultValue. Otherwise, the interpolation
> algorithm below applies. The zero-coordinate rule does not change the
> per-axis or region scalars used in the exempt processing.

In subclause **7.3.1.3, Processing**, replace the paragraph beginning “Each
pair of consecutive AxisValueMap records” with:

> Each pair of consecutive AxisValueMap records for a given axis defines
> a segment and a mapping from initial normalized coordinates to
> segment-mapped coordinates. Within a segment, coordinate values are
> interpolated linearly. The segment-mapped coordinates are final for
> 'avar' version 1. For version 2, they are intermediate coordinates to
> which step 3 below applies. For details of the segment-mapping process
> and an example, see 7.1.4.

In subclause **7.3.1.3, Processing**, after step 3 of the normalization
process and before “In more detail, step 3 proceeds as follows”, insert the
following paragraph:

> When itemVariationStoreOffset is nonzero, computation of the coordinate
> adjustments in an 'avar' version 2 table shall not be omitted solely
> because all intermediate coordinates obtained in step 2 are zero.
> A region with no active axes has an overall scalar of 1 and can therefore
> supply a constant coordinate adjustment. As a result, default user-scale
> axis values can map to nonzero final normalized coordinates. The
> zero-coordinate rule in 7.1.7 is applied to subsequent font-data variation
> processing using the final coordinates obtained in step 3.

In subclause **7.3.10.5, Processing of Variable Composite Glyphs**, insert
the following as the second paragraph:

> When a glyph is loaded from VARC, processing of its VARC record shall
> not be omitted solely because all normalized coordinates are zero.
> This applies also to recursively loaded VARC components and includes
> evaluation of conditions and variation adjustments to component axis
> values and transforms. When a component outline is loaded from 'glyf',
> GLYF, or CFF2, the zero-coordinate rule in 7.1.7 is applied using the
> component's effective normalized coordinates after application of its
> axis values and axis-value deltas. These coordinates can differ from the
> normalized font coordinates.
