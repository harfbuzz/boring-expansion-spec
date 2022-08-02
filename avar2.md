# `avar`—Axis Variations Table Version 2

The axis variations table (`avar`) is an optional table used in variable fonts. Version 1 of the `avar` can be used to modify aspects of how a design varies for different instances along a particular design-variation axis. Version 2 as proposed here allows modifying the design-space across multiple design-variation axes simultanously.

# Table format

Axis variation table:

| Type	| Name	| Description |
| ------|-------|-------------|
| uint16	| majorVersion	| Major version number of the axis variations table — set to 2.
| uint16	| minorVersion	| Minor version number of the axis variations table — set to 0.
| uint16	| <reserved>	| Permanently reserved; set to zero.
| uint16	| axisCount	| The number of variation axes for this font. This must be the same number as axisCount in the 'fvar' table.
| SegmentMaps	| axisSegmentMaps[axisCount]	| The segment maps array — one segment map for each axis, in the order of axes specified in the 'fvar' table.
| Offset32To<DeltaSetIndexMap> | axisIdxMap ||
| Offset32To<ItemVariationStore> | varStore ||


