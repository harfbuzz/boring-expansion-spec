# `avar`—Axis Variations Table Version 2

The axis variations table (`avar`) is an optional table used in variable fonts. Version 1 of `avar` modifies aspects of how a design varies for different instances along a particular design-variation axis. It does this by piecewise linear remapping on a per-axis basis, with certain restrictions. See the [OpenType specification](https://docs.microsoft.com/en-us/typography/opentype/spec/avar) for details.

Version 2, as proposed here, allows changes in one design-variation axis to affect many other design-variation axes, and conversely for one design-variation axis to be affected by many other design-variation axes.

In many cases, `avar` version 2 data does not offer brand new functionality but allows desirable functionality to be implemented much more efficiently by:

* reducing data footprint;
* easing source maintenance by avoiding redundant data;
* making fonts easier for users to control.

Use cases include:

* warped variable font designspaces to reflect a typeface designer’s intention accurately;
* parametric fonts with intuitive control methods and a much reduced data footprint;
* offering simpler methods of control for specialized variable fonts.

# Table format

Axis variation table:

| Type	| Name	| Description |
| ------|-------|-------------|
| `uint16`	| `majorVersion`	| Major version number of the axis variations table — set to 2. |
| `uint16`	| `minorVersion`	| Minor version number of the axis variations table — set to 0. |
| `uint16`	| `<reserved>`	| Permanently reserved; set to zero. |
| `uint16`	| `axisCount`	| The number of variation axes for this font. This must be the same number as axisCount in the 'fvar' table. |
| `SegmentMaps`	| `axisSegmentMaps[axisCount]`	| The segment maps array — one segment map for each axis, in the order of axes specified in the 'fvar' table. |
| `Offset32To<DeltaSetIndexMap>` | **`axisIdxMap`** | Offset from beginning of the table, to optional DeltaSetIndexMap storing variation index mapping. |
| `Offset32To<ItemVariationStore>` | **`varStore`** | Offset from beginning of the table, to optional ItemVariationStore storing variations. |

The table format for `avar` table version 2 is the same as `avar` table version 1 followed by two extra members `axisIdxMap` and `varStore`. It is expected that implementations that handle only version 1 with ignore the entire table by checking the `majorVersion` value.

Delta values are in the range [-2,2). However, in the `ItemVariationData` they are stored as their true value left-shifted 14 bits (i.e. multiplied by 16,384) in one or two bytes, as `int16` (16-bit signed integer) or `int8` (8 bit signed integer). The LONG_WORDS flag should not be set. For the purposes of calculating interpolated delta values, implementations of `ItemVariationStore` can use existing methods of handling `int16` and `int8` values, only at the end dividing by 16,384 to get the actual delta value.

# Processing

Processing of an axis value in an `avar` version 2 table happens in 3 stages:

1. Initial normalization to convert user coordinates to initial normalized coordinates in the range [-1,1].
2. Remapping of initial normalized coordinates via the `axisSegmentMaps` of `avar` version 1, providing intermediate coordinates also in the range [-1,1].
3. Adding interpolated deltas to intermediate axis coordinates via `avar` version 2, providing final coordinates also in the range [-1,1].

Step 3 above deserves explaining in more detail. The `varStore` takes as its input the position in the variation space defined by the set of intermediate axis coordinates obtained in step 2. Each `ItemVariationData` structure in `varStore`, thanks to its regions interacting with the position in variation space, can be assigned a scalar, so that an interpolated delta may be calculated for each delta value. Each interpolated delta is added to the intermediate value of a particular axis, the axis index being defined by `axisIdxMap`. Delta values are encoded as if they are integer values although they represent values smaller by a factor of 2^14, compatible with the F2DOT14 format for normalized axis coordinates (i.e. 1.0 is represented as the integer 16384 and -1.0 as -16384). After the summing operations for all `ItemVariationData` structures, axis values are clamped to to the range [-1,1], giving final coordinates.

The set of final axis coordinates obtained by step 3 is used in the standard variation process described in [Algorithm for Interpolation of Instance Values](https://docs.microsoft.com/en-us/typography/opentype/spec/otvaroverview#algorithm-for-interpolation-of-instance-values), applying to all `gvar` and `ItemVariationStore` data in the font (except that in the `avar` table itself).

The following algorithm implements step 3 above, producing the final normalized axis coordinates:

```c++
    // let coords be the vector of current normalized coordinates.

    std::vector<int> out;
    for (unsigned i = 0; i < coords.size(); i++)
    {
      int v = coords[i];
      uint32_t varidx = i;
      
      if (axisIdxMap != 0)
        varidx = (this+axisIdxMap).map(varidx);
      
      float delta = 0;
      
      if (varStore != 0)
        delta = (this+varStore).get_delta (varidx, coords);

      v += std::roundf (delta);
      v = std::clamp (v, -(1<<14), +(1<<14));

      out.push_back (v);
    }
    for (unsigned i = 0; i < coords.size(); i++)
      coords[i] = out[i];
```

## Inverse avar processing

Normally it is not possible for an application to obtain normalized axis values directly, these being private to the font engine. In fonts with an `avar` version 2 table, however, it can be useful to know the axis values that would invoke a given instance if the font engine did not support `avar` version 2. Use cases for this data include:

* informing users of the effective settings of parametric axes;
* providing a polyfill for `avar` version 2.

The solution is for the application to implement the `avar` algorithms, both version 1 and version 2. This requires not only access to the binary `avar` table, but also knowledge of the font’s axis extents as defined in the `fvar` table. Once an application has a complete set of final normalized values as defined above, the inverse of the `avar` version 1 algorithm can be applied to obtain effective user values for all axes. (Care must be taken to avoid a divide-by-zero error in implementing an inverse `avar` version 1 mapping, in the case where consecutive `toCoordinate` values are identical.)

Almost all fonts can provide a valid set of user values for a given set of final normalized values. Exceptions are those fonts that encode deltas in a region that is not accessible without `avar` version 2, namely those fonts where the `fvar` definition of an axis normally precludes an active negative region or a active positive region by having its default equal to its minimum or its maximum respectively. Such cases are expected to be rare.


# Discussion

This table uses the same variation mechanism used in multiple other places in OpenType 1.8, to vary normalized axis values themselves. In particular, variation deltas are stored in an `ItemVariationStore`, and the variation indices are stored in a `DeltaSetIndexMap`.

The processing is broken into two steps: let's call them the v1 and v2 steps. The v1 step is the same as an `avar` version 1 table. The v2 step is the algorithm listed above. The `ItemVariationStore` is driven with the normalized axis values generated by the v1 step, to produce deltas to add to those same normalized axis values, to produce the final normalized axis values. It is this step that allows arbitrary multi-axis transformation, because unlike the v1 step, the delta for, and hence adjustment to, each axis can be affected by the input value of every axis.

## Hidden axes

Since many `avar` version 2 fonts have axes not not intended for manual adjustment, it is recommended that such axes set the “hidden” flag in `VariationAxisRecord` of the [`fvar`](https://docs.microsoft.com/en-us/typography/opentype/spec/fvar) table.


# Construction

The way `ItemVariationStore`s are built is typically by using a variation modeler, that takes a series of _master_ values at certain locations in the design-space, and produces delta-sets to be stored in an `ItemVariationStore`. This usage is no different. To build the `avar` version 2 mapping tables, the designer will need to produce a mapping of input axis locations and their respective output axis locations. This data then will constitute the set of _masters_ to be fed to the variation modeler and populate the `ItemVariationStore` that will go into the `avar` version 2 table. The variation index for each axis will be stored in `axisIdxMap`.

# Use cases

## 1. Designspace warping

In the OpenType variations specification, [registered axes](https://docs.microsoft.com/en-us/typography/opentype/spec/dvaraxisreg) offer a standard way to offer users control of a variable font on reasonably well defined scales. For example, users learn that the Regular version of a font has Weight=400, Width=100, while the Bold has wght=700, wdth=100. A typical 2-axis variable font with `wght` and `wdth` axes might have the following nine Named Instances:


|  | **75** | **100** | **125** |
| --- |---|---|---|
| **300**   | Light Condensed | Light | Light Extended |
| **400** | Condensed | Regular | Extended |
| **700**    | Bold Condensed | Bold | Bold Extended |

However, in fonts with more than one design axis, this approach lacks the flexibility of older methods of interpolation, where the type designer used a font design application to specify freely the axis values for, say, the Bold Condensed. Notably, the `wght` coordinate of Bold Condensed did not need to match that of the Bold, nor did the `wdth` coordinate need to match that of the Condensed. A Bold Condensed with `wght`,`wdth` of (677,81) rather than (700,75) was perfectly reasonable. Our grid of instances can then look something like this:

|  | **Condensed** | **Regular** | **Expanded** |
| --- |---|---|---|
| **Light**   | 300,75 | 300,100 | 300,115 |
| **Regular** | 400,75 | 400,100 | 400,120 |
| **Bold**    | 677,81 | 695,100 | 700,125 |

While with OpenType 1.8 it is possible to set up a designspace like this, it has the significant drawback that many applications and systems expect all Bold weights to be at `wght`=700 and all Condensed weights to be at `wdth`=75, whatever the values of other axes. Thus, in practice, many OpenType 1.8 fonts encode numerous additional masters that ensure the design is as intended at all designspace locations. This not only impacts the data footprint significantly, but also adds to the maintenance burden of the font.

Furthermore, there are many cases where a type designer does not intend to offer instances at certain combinations of the axis extremes. For example, a Black Condensed instance is often difficult to design, and many type families omit it. Desired behaviour may be for the font to provide the same instance for the Black Condensed as it does for the Bold Condensed. [A 2019 discussion in the Glyphs forum](https://forum.glyphsapp.com/t/variable-with-2-axis-in-6-masters/11958/3) summarizes the problem, and again comes up with the only possible solution of data duplication:

> *PascalZoghbi:* [describing the intended result] The weight axes are between 100 and 900 in the Normal and Wide master, but it only goes up to 700 in the Condensed master. …  
> *mekkablue:* No matter what you do, Glyphs 2 requires a rectangular master setup. I would post-process the file with TTX, and insert an avar table.  
> *PascalZoghbi:* Duplicating the Bold 700 and making it Black 900 solved the problem for now. The quick easy fix. The problem is that it makes the file heavier.

Luc(as) de Groot analyses the same problem in his [Typo Labs 2018 talk](https://www.youtube.com/watch?v=I75Efo7whrs&t=35m48s), where he calls for “virtual masters”.

Using `avar` version 2 we can apply deltas to axis values anywhere in the designspace, thus “warping” it to resolve the issues described. As with variation deltas in general, we also benefit from interpolation when axis values are influenced by a delta but are not exactly at the location of the delta. Each delta value is encoded as F2DOT14. Its interpolated value is determined by the standard OpenType variations algorithm, then added to the value obtained from the standard normalization process. Thus, assuming the Bold Condensed instance at (700,75) has normalized coordinates (1,-1) and assuming the designspace location (677,81) has normalized coordinates ((677-400)/(700-400), -1 + (81-75)/(100-75)) = (0.9233,-0.76), then the `ItemVariationStore` needs a delta set with peaks at (-1,1), thus with region ((-1,-1,0),(0,1,1)) and the following delta values:

* `wght` -0.0767
* `wdth` +0.24

In practice there may be other delta sets required at the partial default locations (-1,0) and (0,1), which then influence the delta values required at (-1,1), in line with normal variations math.

The result of implementing designspace warping this way is that we preserve the original axis values where applications and systems expect them, namely all Bold instances at `wght`=700 and all Condensed weights at `wdth`=75, while at the same time minimizing the data footprint.


## 2. Duplication of axis values for non-linear interpolation

Certain variable fonts encode delta sets in such a way that, when multiple axes are synchronized, outline points move in curves rather than in straight lines. This technique was [invented by Underware](https://underware.nl/case-studies/hoi/) and [presented in 2018](https://www.youtube.com/watch?v=YVMukTiElUQ) under the name HOI (higher order interpolation). The details of the non-linear encoding need not concern us here, but a practical drawback under OpenType 1.8 is that it requires users to set multiple design axes to identical values in order to obtain valid instances. When axes are not synchronized, the resulting glyphs are severely distorted and useless. Synchronization using JavaScript has been proposed as a solution. A clever exploit of the CFF2 format, allowing a single axis to perform all the necessary non-linear adjustment of outline points, has also been demonstrated (but this method cannot be adapted to adjust delta values in an `ItemVariationStore` non-linearly).

The `avar` version 2 table provides a solution to the synchonization problem, such that a user adjusts one “primary” axis and that value is cloned to other subordinate axes. Ideally the subordinate axes have the Hidden flag set in `fvar` to discourage manual operation. For a font with 3 axes requiring identical values, if the first axis is considered primary, then the other two can encode delta sets in `avar`’s `ItemVariationStore` to clone the value of the first axis. Assuming the subordinate axes have the same minimum, default and maximum values as the primary axis, each subordinate axis requires either one or two delta sets to clone the negative and positive regions of the normalized value of the primary axis:

* If the axes use only the normalized region [0,1] then one delta set per subordinate axis is needed, referring to the region (0,1,1) of the primary axis and having the delta value of 1. This is the common case.

* If the axes use only the normalized region [-1,0] then one delta set per subordinate axis is needed, referring to the region (-1,-1,0) of the primary axis and having the delta value of -1.

* If the axes use both the normalized regions [-1,0] and [0,1] then two delta sets per subordinate axis are needed, one referring to the region (0,1,1) of the primary axis and having the delta value of 1, the other referring to the region (-1,-1,0) of the primary axis and having the delta value of -1.


## 3. Simplified controls for parametric fonts without redundant data

For many years, parametric fonts have promised both extreme flexibility and a compact data footprint. The [Type Network Variations Proposal](https://variationsguide.typenetwork.com) of 2017, explored in the fonts [Amstelvar](https://github.com/googlefonts/amstelvar), [Roboto Flex](https://github.com/googlefonts/roboto-flex) and others, demonstrates the potential for usable parametric fonts with large character sets. That flexibility comes from a designspace of many axes – sometimes 10 or more – which naturally raises issues of control. Faced with 10 sliders, users are unlikely to be able to find the instance they want, and indeed will very likely get “lost” in a unusable region of the designspace.

In OpenType 1.8, the solution to getting “lost” is to include “blended” axes that combine the deltas of parametric axes in such a way that they function in exactly the same way as they would in a non-parametric font. The blended axes are exposed using the registered axes of `wght`, `wdth`, `opsz`, etc., or well specified custom axes, and they are encoded in the standard OpenType 1.8 manner using `gvar` and related data. The major problem with such fonts is their large data footprint, as the blended axes effectively replicate much of what the parametric axes do. For a font containing both parametric and blended axes, the problem is particularly acute, so the parametric axes may be omitted — thereby removing core functionality. In such cases the parametric nature of the font is thus reduced to a convenience of sources.

The `avar` table version 2 provides a method to deploy a purely parametric font with user-friendly axes, while avoiding the data bloat of blended axes. The user axes, for example `wght` and `wdth`, on their own do nothing (i.e. they have no `gvar` or similar data), but they control the parametric axes by means of `avar`. The blending of parametric values into user-facing values effectively happens live in the font engine. The parametric axes are expected to have the Hidden flag set in `fvar` to discourage (but not block) manual operation.

