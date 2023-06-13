# boring-expansion-spec

## What do we want to change?

1. Break the 65k glyph limit on glyphs in a single font file
   * [why?](#1-break-the-65k-glyph-limit)
   * [spec proposal](./beyond-64k.md)
   * [test fonts](https://drive.google.com/drive/folders/1iRBebQS5z0bjG3EBGZ53ZB6TOBg3LhUb)
1. Enable cubic curves in `glyf` outlines
   * [why?](#2-enable-cubic-curves-in-glyf-outlines)
   * [spec proposal](./glyf1-cubicOutlines.md)
   * [test fonts](https://drive.google.com/drive/folders/1SdoQYRz-4K9x5mB5LxGBFYmj9UpF_MOf)
1. Enable components to take advantage of variations
   * [why?](#3-enable-components-to-take-advantage-of-variations)
   * [spec proposal](./glyf1-varComposites.md)
   * [test fonts](https://github.com/googlefonts/smarties/tree/main/fonts)
1. Enhance core variation capability
   1. Improve support for user-facing vs internal axis distinction and the mapping between them
      * [why?](#4i-user-facing-vs-internal-axes)
      * [spec proposal](./avar2.md)
      * [test fonts](https://drive.google.com/drive/folders/1OjZFadiSKwzPnGWWCmNwscJN8bsjfqR0)
   1. Explicitly support non-linear interpolation
      * [why?](#4ii-explicitly-support-non-linear-interpolation)
      * _no spec proposal so far_
   1. Efficient storage of sparse variation data
      * [why?](#4iii-efficient-storage-fo-sparse-variation-data)
      * _no spec proposal so far_

## Why?

The proposed changes, taken as a whole, allow us to create compact pan-Unicode fonts made up of
reusable parts that are built using enhanced variation capabilities. Further, the designer is
empowered to separate how the parts are crafted and assembled from how they are presented to the user.

### 1. Break the 65k glyph limit

See https://rsheeter.github.io/more_gids.

### 2. Enable cubic curves in `glyf` outlines

The input to a font is often from a design tool that uses cubic curves. Quadratic approximation reduces quality and increases filesize. Explicit support for cubic curves in `glyf` resolves both
issues.

### 3. Enable components to take advantage of variations

Design tool features like [smart components](https://glyphsapp.com/learn/smart-components) allow
designers to build fonts out of reusable parts. For CJK we've seen experiments where tens of
thousands of glyphs are built from at least an order of magnitude fewer parts. 

By enabling reuse of a glyph at a different position in designspace the font binary can
leverage this to significantly reduce filesize. See prior art in
[Variable Components for COLRv1](https://github.com/googlefonts/colr-gradients-spec/issues/277).

This can be done by leveraging `COLR` or by enhancing `glyf`.

### 4. Enhance core variation capability

#### 4.i user-facing vs internal axes

Reduce complexity in the design of variable fonts and produce more compact files.

User-facing axes (weight, width, etc) are currently the same axes the designer uses. However,
these axes are not naturally orthogonal. Designers are forced to resolve this by introducing
additional masters, which costs the designer time and the end user bytes. A better system would
enable user-facing axes to be assembled from other axes, which could (and should) be orthogonal.

The Type Network [axis set](https://variationsguide.typenetwork.com/) is a relatively well known example.

To assemble a set of non-orthogonal user facing axes from a set of orthogonal axes requires a
non-linear many:many mapping. `avar` provides a piecewise linear mapping from an input position
on a single axes to a different position on that axes. This is inadequate; a new and more powerful
mapping mechanism is required.

#### 4.ii Explicitly support non-linear interpolation

It's clear non-linear interpolation (NLI) is desired and, perhaps unintentionally, possible due to being
able to provide multiple scalars which are multiplied together. See [NLI in Variable Fonts](https://github.com/PeterConstable/OT_Drafts/blob/master/NLI/UnderstandingNLI.md).

Today NLI is complex and not necessarily size-efficient. Add additional basis functions
to support NLI, clean mapping of an axis value to rotation, and extrapolation.

#### 4.iii Efficient storage fo sparse variation data

If, as proposed, we add variation aware components and stronger mapping between user-facing and internal axes it becomes increasingly desirable to have larger numbers of axes, each performing a narrow task. Currently use of an axis comes with a huge penalty: variation stores encode positions
for all axes rather than only the axes that contribute to the result.

Upgrade the variation stores to only consume bytes for axes that influence the result.

## ISO Submissions

* WG03_otf-improvements ([docx](iso_docs/WG03_otf-improvements.docx?raw=true), [pdf](iso_docs/WG03_otf-improvements.pdf?raw=true))
   * MPEG document identifier: m62947, submitted April 2023
   * Initial submission to spur ISO discussion

## References

1. [Better Engineered Font Formats](https://docs.google.com/presentation/d/1dVfuU7YhUBXg9MtU6kYBXVs9082PiHpwhGPYa--yA7c/edit?usp=sharing)
   * Notes from initial presentation
