# `glyf`—Glyf Data Table Version 1 / Variable Components

The glyf data table ([`glyf`](https://docs.microsoft.com/en-us/typography/opentype/spec/glyf)) stores the glyph outline data.  The version number for this table is stored in the [`head`](https://docs.microsoft.com/en-us/typography/opentype/spec/head) table's `glyphDataFormat` field.  The only currently defined version of the `glyf` table is version 0.  In this proposal we propose version 1 of the `glyf` table to add the following features:

* Variable Components.

## Variable Components

This proposal is mainly based on BlackFoundry's [Variable Components](https://github.com/BlackFoundryCom/variable-components-spec) and [Storing Variable Components in UFO files](https://github.com/BlackFoundryCom/variable-components-in-ufo).


