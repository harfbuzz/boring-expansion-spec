# DMAP lookup fallback

The `DMAP` table shall be consulted before `cmap`. If a `DMAP` lookup has no
matching mapping or maps to glyph ID 0, the corresponding `cmap` lookup shall
be performed. This applies to ordinary character mappings and non-default
Unicode variation sequence mappings in formats 14 and 15.

A match in a default UVS range in a format 14 or 15 `DMAP` subtable is a
successful match. The glyph shall be obtained by ordinary character mapping,
consulting `DMAP` before `cmap`. The `cmap` UVS mapping shall not be consulted.
