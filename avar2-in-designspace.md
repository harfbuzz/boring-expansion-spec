# designspace with avar2

This document discusses the representation of the `avar` version 2 table (“avar2”) in .designspace documents.

It is not immediately clear what kind of representation is suitable for avar2. It is represented using the *ItemVariationStore* structure, which enables full OpenType Variations math. In other words, the avar2 definition allows arbitrary regions to be created in a designspace, in which scalars are calculated depending on user-specified location, those scalars then being multiplied a delta value which is then added to the location of a single axis; typically these deltas come in “delta sets”, where it is specified which delta applies to which axis location. *There is currnetly no representation for .designspace that represents arbitrary OTVar math.*

This flexibility of full OTVar may be contrasted with the limited representation of variation of glyph outline points in a .designspace document, where a set of masters is specified, each with its designspace location, the compiler determining the deltas.

A further difference between ItemVariationStore and .designspace is that deltas are in normalized 2.14 coordinates rather than 16.16 user coordinates. If a .designspace document is intended to be readable by humans, normalized coordinates are a poor choice, and even deltas of any kind are novel compared with the absolute values of `<source><location><dimension>`.

## Goals

Try to make the avar2 representation similar-ish to avar1.

A significant goal I am assuming is to try to represent avar2 in absolute user coordinates, so no explicit delta values and no normalized coordinates.

One thing that should definitely *not* be in .designspace is multiple ItemVariationData objects. Splitting data into multiple ItemVariationData objects is a compiler implementation detail.

The avar2 use-case classes already identified (distortion, parametric, HOI, fences) are not representable in a “masters” style. We need to allow the min, peak, max style of variation region specification. It seems therefore that these two approaches to the representation of variation need to be somehow combined.

## Representation
Here follow some suggestions for how to represent the distortion, parametric and HOI use-cases classes in .designspace. We start with a sample avar1 representation for reference.

Note that, in avar1, the `<map>` element is within an `<axis>` element. For avar2, we bring the `<map>` element up to the same level as the `<axis>` elements. A .designspace document representing avar2 data has a single `<map>` element, containing one or more `<region>` elements.

It is recommended to allow reference to axes by tag rather than by name.

### avar1 representation

```xml
<axes>
    <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400">
      <map input="200" output="220"/>
      <map input="600" output="615"/>
      <map input="700" output="720"/>
    </axis>
</axes>
```

### avar2 use case: distortion

```xml
<axes>
  <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400"></axis>
  <axis tag="wdth" name="Width" minimum="50" maximum="200" default="100"></axis>

  <map>
    <region>
      <input>
        <dimension name="Weight" xvalue="700" />
        <dimension name="Width" xvalue="150" />
      </input>
      <output>
        <dimension name="Weight" xvalue="650" />
        <dimension name="Width" xvalue="140" />
      </output>
    </region>
  </map>
</axes>
```

To determine delta values, the compiler normalizes input xvalue and output xvalue to 2.14 using the normal avar1 process, then subtracts the normalized input value from the normalized output value. If there an output for an axis but no input, the axis default value is used. Normalizing input and output xvalues gives:
* `wght` input: 700 → (700-400)/(900-400) = 0.6
* `wght` output: 650 → (650-400)/(900-400) = 0.5
* `wdth` input: 150 → (150-100)/(200-100) = 0.5
* `wdth` output: 140 → (140-100)/(200-100) = 0.4

Subtracting the normalized input value from the normalized output value gives:
* `wght`: 0.5 - 0.6 = -0.1
* `wdth`: 0.4 - 0.5 = -0.1

So for the region defined by `wght` [400, 700, 900], `wdth` [100, 150, 200] we have a delta set of [-0.1, 0.1] for the axes `wght` and `wdth`.

This region covers the whole “upper right quadrant” of the designspace, so we did not need to specify min and max. (If xvalue > default, we use axis default and axis max as dimension min and max; if xvalue < default, we use axis min and axis default as dimension min and max.) To be explicit and to demonstrate the use of min and max, we could have written the following to define exactly the same region and therefore the same action for the compiler:


```xml
  <map>
    <region>
      <input>
        <dimension name="Weight" min="400" xvalue="700" max="900"/>
        <dimension name="Width" min="100" xvalue="150" max="200"/>
      </input>
      <output>
        <dimension name="Weight" xvalue="650" />
        <dimension name="Width" xvalue="140" />
      </output>
    </region>
  </map>

```

### avar2 use case: parametric

This .designspace is for a parametric font with 3 user axes and 5 hidden parametric axes. We show a single mapping for the Weight axis acting on 3 parametric axes. Exactly the same method is used by the compiler as described above. The key difference with parametrics is that, since the output axis values are not specified in the input, the subtraction operates using the default values instead.

For clarity only a single region is represented, that being from default to max `wght`. A full parametric font would need “monovar” regions for both sides of default for all 3 user axes, and optionally “duovar” and “trivar” regions for the combinations of user axes (3^3-1 = 26 regions in all for a 3-axis font, assuming no intermediate regions).

```xml
<axes>
  <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400"></axis>
  <axis tag="wdth" name="Width" minimum="50" maximum="200" default="100"></axis>
  <axis tag="opsz" name="Optical Size" minimum="6" maximum="72" default="12"></axis>
  <axis tag="XOPQ" name="XOPQ" minimum="18" maximum="263" default="176" hidden="1"></axis>
  <axis tag="XTRA" name="XTRA" minimum="324" maximum="640" default="562" hidden="1"></axis>
  <axis tag="YOPQ" name="YOPQ" minimum="15" maximum="132" default="124" hidden="1"></axis>
  <axis tag="YTUC" name="YTUC" minimum="500" maximum="1000" default="750" hidden="1"></axis>
  <axis tag="YTLC" name="YTLC" minimum="420" maximum="570" default="500" hidden="1"></axis>

  <map>
    <region>
      <input>
        <dimension name="Weight" xvalue="900" />
      </input>
      <output>
        <dimension name="XOPQ" xvalue="260" />
        <dimension name="XTRA" xvalue="370" />
        <dimension name="YOPQ" xvalue="132" />
      </output>
    </region>
  </map>

</axes>
```

As with the calculation of outline variation deltas from master outline coordinates, we need to recall that deltas are based on accumulated values at the “corners”. This may be difficult to generalize given a) the deltas are ultimately in 2.14, and b) we want to allow arbitrary regions, not just orthogonal ones.


### avar2 use case: HOI

This .designspace is for a HOI font with 1 user axis and 2 hidden subservient axes.

```xml
<axes>
  <axis tag="HOI0" name="Animate" minimum="0" maximum="1000" default="0"></axis>
  <axis tag="HOI1" name="HOI1" minimum="0" maximum="1000" default="0" hidden="1"></axis>
  <axis tag="HOI2" name="HOI2" minimum="0" maximum="1000" default="0" hidden="1"></axis>

  <map>
    <region>
      <input>
        <dimension name="Animate" xvalue="1000" />
      </input>
      <output>
        <dimension name="HOI1" xvalue="1000" />
        <dimension name="HOI2" xvalue="1000" />
      </output>
    </region>
  </map>

</axes>
```


## Further notes

It will likely be a common occurence that we wish to add avar2 table to a compiled font. Such fonts will not have any UFO data to work with, so if a .designspace document is used to generate the avar2 table, we’ll need to allow it to have an empty <source> section, or for the <source> section to contain a reference to a compiled font file.

Mappings for avar1 may be combined with avar2. The avar1 maps are within each `<axis>` element, the avar2 maps are at the same level. If an avar2 map was representable as avar1, the compiled would be free to use that option.

We might also consider the following:
* a relative mode, in case you want to work in deltas, with suggested representations within `<output><dimension>`:
  * `mode="absolute"` (default), `mode="relative"` (deltas, optional)
  * use `xdelta` instead of `xvalue`
* allow direct specification of normalized coordinates

## Comparison of other XML methods of representing/implying variation math

### TTX gvar

```xml
<tuple>
  <coord axis="TIME" min="0.0" value="0.06665" max="0.13336"/>
  <delta pt="0" x="-10" y="4"/>
  <delta pt="1" x="-14" y="5"/>
  <delta pt="2" x="17" y="9"/>
  <delta pt="3" x="-7" y="7"/>
  <delta pt="4" x="-11" y="0"/>
  <delta pt="5" x="-11" y="0"/>
  ...
</tuple>
```

### TTX ItemVariationStore
...


### TTX CFF2 blend

...


### designspace source::location

...

