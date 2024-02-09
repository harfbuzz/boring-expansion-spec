**INTERNATIONAL ORGANISATION FOR STANDARDISATION**

**ORGANISATION INTERNATIONALE DE NORMALISATION**

**ISO/IEC JTC 1/SC 29/WG 3**

**CODING OF MOVING PICTURES AND AUDIO**

**ISO/IEC JTC 1/SC 29/WG 3 m** **NNNN**

**Rivendell – February 2024**

**Title: Updates based on Public Feedback to Changes to the OFF Font
Format**

**Author: Dave Crossland (Google Inc., dcrossland@google.com), Behdad
Esfahbod (behdad@behdad.org), Laurence Penney (lorp@lorp.org), Liam Quin
(Delightful Computing, liam@delightfulcomputing.com), Rod Sheeter
(Google Inc., rsheeter@google.com)**

# <span id="anchor"></span>Introduction

This introduction is to give context and is not itself proposed text.

A recent proposal to the ISO MPEG OpenFont committee,
[m66260](https://github.com/harfbuzz/boring-expansion-spec/blob/main/iso_docs/WG03-beyond-64k-glyphs-2024-01e.pdf),
was accepted in January of 2024. Since that time there have been a
number of comments on the proposal.

Some of these pointed out small typographical errors, and these are
included in this Proposal.

Some of the comments made technical suggestions, and, where appropriate,
these are also incorporated in this proposal.

The primary goal is to enable font producers to create fonts containing
more than 65535 glyphs, and to guide implementors, technical writers,
trainers, and others, in their use.

Technical discussion of most the items proposed here may be found in
Github issues, as noted for each change.

# hdmx

[](https://github.com/harfbuzz/boring-expansion-spec/issues/131)

<https://github.com/harfbuzz/boring-expansion-spec/issues/131>

In 5.6.2 hdmx—Horizontal device metrics, in the Device Record table
headed Each DeviceRecord for format 0 looks like this, change the table
as follows:

Each DeviceRecord for format 0 looks like this.

|               |                     |                                                                                    |
|---------------|---------------------|------------------------------------------------------------------------------------|
| Device Record |                     |                                                                                    |
| Type          | Name                | Description                                                                        |
| uint8         | pixelSize           | Pixel size for following widths (as ppem).                                         |
| uint8         | maxWidth            | Maximum width.                                                                     |
| uint8         | widths\[numGlyphs\] | Array of widths (numGlyphs is from the 'MAXP' table if present, otherwise ‘maxp’). |

# LTSH

<https://github.com/harfbuzz/boring-expansion-spec/issues/132>

In 5.6.4 LTSH—Linear threshold, after “The format for the table is:”,
change the table as follows:

The format for the table is:

|        |                    |                                                                                                    |
|--------|--------------------|----------------------------------------------------------------------------------------------------|
| Type   | Name               | Description                                                                                        |
| uint16 | version            | Version number (starts at 0).                                                                      |
| uint16 | numGlyphs          | Number of glyphs (numGlyphs is from the 'MAXP' table if present, otherwise ‘maxp’).                |
| uint8  | yPels\[numGlyphs\] | The vertical pel height at which the glyph can be assumed to scale linearly. On a per glyph basis. |

# BASE

<https://github.com/harfbuzz/boring-expansion-spec/issues/129>

# 

In 6.3.1.4 BASE table structure, update BaseCoordFormat3=4 table to fix
a typo: format should be 4, not 2.

*BaseCoordFormat*4* table: Design units plus contour point *(24-bit
glyph ID)**

|        |                 |                                               |
|--------|-----------------|-----------------------------------------------|
| Type   | Name            | Description                                   |
| uint16 | baseCoordFormat | Format identifier – format = 4                |
| int16  | coordinate      | X or Y value, in design units                 |
| uint24 | referenceGlyph  | Glyph ID of control glyph                     |
| uint16 | baseCoordPoint  | Index of contour point on the reference glyph |

****

# JSTF

In 6.3.5.1 JSTF—The justification table, after JsfScriptRecord and
before Justification script table, insert the following new subsection,
just after “Example 1 at the end of this clause shows a JSTF Header
table and JstfScriptRecord.”

**JSTF header **1.1**

|                   |                                       |                                                                     |
|-------------------|---------------------------------------|---------------------------------------------------------------------|
| Type              | Name                                  | Description                                                         |
| uint16            | majorVersion                          | Major version of the JSTF table, = 1                                |
| uint16            | minorVersion                          | Minor version of the JSTF table, = 1                                |
| uint16            | jstfScriptCount                       | Number of JstfScriptRecords in this table                           |
| JstfScriptRecord  | jstfScriptRecords\[jstfScriptCount\]  | Array of JstfScriptRecords, in alphabetical order by jstfScriptTag  |
| uint16            | jstfScriptCount2                      | Number of JstfScriptRecords2 in this table                          |
| JstfScriptRecord2 | jstfScriptRecords2\[jstfScriptCount\] | Array of JstfScriptRecords2, in alphabetical order by jstfScriptTag |

**JstfScriptRecord**2**

|          |                  |                                                            |
|----------|------------------|------------------------------------------------------------|
| Type     | Name             | Description                                                |
| Tag      | jstfScriptTag    | 4-byte JstfScript identification                           |
| Offset32 | jstfScriptOffset | Offset to JstfScript2 table, from beginning of JSTF Header |

****

**After the JstfScript table, add:**

****The JstfScript2 table is based on the JstfScript table, but has
32-bit offsets:****

**JstfScript**2* table *

<table>
<tbody>
<tr class="odd">
<td>Type</td>
<td>Name</td>
<td>Description</td>
</tr>
<tr class="even">
<td>Offset32</td>
<td>extenderGlyphOffset</td>
<td>Offset to ExtenderGlyph table, from beginning of JstfScript table
(may be NULL)</td>
</tr>
<tr class="odd">
<td>Offset32</td>
<td>defJstfLangSysOffset</td>
<td>Offset to Default JstfLangSys table, from beginning of JstfScript2
table (may be NULL)</td>
</tr>
<tr class="even">
<td>uint16</td>
<td>jstfLangSysCount</td>
<td>Number of JstfLangSysRecords in this table, may be zero (0)</td>
</tr>
<tr class="odd">
<td>JstfLangSysRecord</td>
<td>jstfLangSysRecords<br />
[jstfLangSysCount]</td>
<td>Array of JstfLangSysRecords, in alphabetical order by
jstfLangSysTag</td>
</tr>
</tbody>
</table>

**

After **the Extender Glyph table, just before Justification Language
System table, insert:**

****

**ExtenderGlyph**2* table*

**The *ExtenderGlyph2* table supports fonts containing more than 65535
glyphs.**

|        |                              |                                                    |
|--------|------------------------------|----------------------------------------------------|
| Type   | Name                         | Description                                        |
| uint32 | glyphCount                   | Number of extender glyphs in this script           |
| uint32 | extenderGlyphs\[glyphCount\] | Extender glyph IDs – in increasing numerical order |

# FVAR

[](https://github.com/harfbuzz/boring-expansion-spec/issues/15)

<https://github.com/harfbuzz/boring-expansion-spec/issues/15>

In 7.3.3 fvar—Font variations table, after VariationAxisRecord, after
the paragraph about the HIDDEN_AXIS tag, add a new final paragraph as
follows:

For smooth animation, and for non-linear interpolation, it may be
necessary for a font to use multiple axes with the same axisTag.

If a font contains more than one axis with the same *axisTag*, at most
one of those axes shall be visible (i.e. have the HIDDEN_AXIS bit set to
zero). The VariationAxisRecord for such a visible axis in this case
shall appear first, before the records for any of the other axes with
that same axisTag, all of which shall have their HIDDEN_AXIS flag set to
1.

> ****

\[end\]
