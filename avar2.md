# `avar`—Axis Variations Table Version 2

The axis variations table (`avar`) is an optional table used in variable fonts. Version 1 of the `avar` can be used to modify aspects of how a design varies for different instances along a particular design-variation axis. Version 2 as proposed here allows modifying the design-space across multiple design-variation axes simultanously.

# Table format

Axis variation table:

| Type	| Name	| Description |
| ------|-------|-------------|
| `uint16`	| `majorVersion`	| Major version number of the axis variations table — set to 2. |
| `uint16`	| `minorVersion`	| Minor version number of the axis variations table — set to 0. |
| `uint16`	| `<reserved>`	| Permanently reserved; set to zero. |
| `uint16`	| `axisCount`	| The number of variation axes for this font. This must be the same number as axisCount in the 'fvar' table. |
| `SegmentMaps`	| `axisSegmentMaps[axisCount]`	| The segment maps array — one segment map for each axis, in the order of axes specified in the 'fvar' table. |
| `Offset32To<DeltaSetIndexMap>` | `axisIdxMap` | Offset from beginning of the table, to optional DeltaSetIndexMap storing variation index mapping. |
| `Offset32To<ItemVariationStore>` | `varStore` | Offset from beginning of the table, to optional ItemVariationStore storing variations. |

 `SegmentMaps` is the same as in `avar` [version 1.0](https://docs.microsoft.com/en-us/typography/opentype/spec/avar#table-formats).

The table format for `avar` table version 2 is the same as `avar` table version 1 followed by two extra members `axisIdxMap` and `varStore`. However, because the version 2 table uses `majorVersion` 2, an older implementation would not be able to interpret a version 2 table even though structurally it begins the same was as a version 1 table.

# Processing

Processing of axis values in a `avar` version 2 table starts off the same as a `avar` version 1 table. That is, axis coordinates are normalized and then mapped according to `axisSegmentMaps`. After that, the following algorithm is applied to produce the final normalized axis coordinates:

```
    hb_vector_t<int> out;
    out.alloc (coords_length);
    for (unsigned i = 0; i < coords_length; i++)
    {
      int v = coords[i];
      uint32_t varidx = varidx_map.map (i);
      float delta = var_store.get_delta (varidx, coords, coords_length, var_store_cache);
      v += roundf (delta);
      v = hb_clamp (v, -(1<<14), +(1<<14));
      out.push (v);
    }
    for (unsigned i = 0; i < coords_length; i++)
      coords[i] = out[i];
```
