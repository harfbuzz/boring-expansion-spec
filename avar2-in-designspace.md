# Proposal for representing avar2 mappings in designspace documents

This document discusses the representation of the `avar` version 2 table (“avar2”) in [.designspace documents](https://github.com/fonttools/fonttools/blob/main/Doc/source/designspaceLib/xml.rst).

## Introduction

*There is currently no representation in .designspace that represents arbitrary variations math.*

This fact means it is not immediately clear what kind of representation is suitable for avar2. In a compiled font, avar2 data uses the *ItemVariationStore* structure, which enables full OpenType Variations math. In other words, the avar2 definition allows arbitrary regions to be created in a designspace, in which scalars (values between 0 and 1) are calculated depending on user-specified location; those scalars are then multiplied by a delta value which is then added to the location of a single axis; typically the deltas come in “delta sets”, where it is specified which delta applies to which axis location.

This flexibility of variation math may be contrasted with the limited representation of variation of glyph outline points in a .designspace document, where a list of sources is specified, each with its designspace location, the compiler determining the deltas.

A further difference between ItemVariationStore and .designspace is that deltas are in normalized 2.14 coordinates rather than int16 user coordinates. If a .designspace document is intended to be readable by humans, normalized coordinates are a poor choice, and even deltas of any kind are novel compared with the absolute values of \<source\>\<location\>\<dimension\>.

## Goals

Allow complete variations math to be specified in .designspace documents in a manner suitable for avar2.

Try to make the avar2 representation similar-ish to avar1, avoiding new terminology if possible.

Represent avar2 in absolute user coordinates, so no explicit delta values and no normalized coordinates. (A significant benefit of absolute coordinates in .designspace is that it is easy to choose a different default source. It is not clear whether this benefit can be retained in avar2 representations.)

One thing that should surely *not* be in .designspace is multiple *ItemVariationData* objects. Splitting varation data into multiple ItemVariationData objects is a compiler implementation detail.

The avar2 use-case classes already identified (distortion, parametric, HOI, fences) are not representable in the “sources/masters” style. We need to allow the min, peak, max style of variation region specification. It seems therefore that these two approaches to the representation of variation need to be somehow combined.

Ideally the representation can be adopted later for full-power variation math on outline points and metrics values.

## avar1 representation (example for reference)

```xml
<axes>
    <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400">
      <map input="100" output="120"/>
      <map input="600" output="650"/>
      <map input="800" output="780"/>
    </axis>
    <axis tag="wdth" name="Width" minimum="50" maximum="200" default="100">
      <map input="70" output="75"/>
      <map input="120" output="135"/>
    </axis>
</axes>
```

## avar2 representation
The proposed syntax is to keep the mappings definition within the \<axes\> element, and reuse the \<map\> element of avar1 by bringing it up to the same level as the \<axis\> elements, and redefining its contents as follows.

1. The \<map\> element contains one or more \<region\> elements.

2. Each \<region\> element contains one \<input\> element and one \<output\> element.

3. The \<input\> element contains one or more \<dimension\> elements and thereby defines a variation region as defined in the OpenType Variations specification. Regions are defined by specifying a subset of axes, and assigning each axis a minimum, a peak and a maximum (the familiar tent function of variation math). The \<dimension\> element specifies an axis and its tent function. In cases where default and an extreme may be implied from a given peak, minimum and maximum may be omitted.

4. The number of \<dimension\> elements within an \<input\> or \<output\> element is often less than the number of axes, as is normal in variation math.

5. The \<output\> element contains one or more \<dimension\> elements, each of which specifies an absolute location (or maybe a relative delta) on one axis in the designspace.

6. The set of \<input\> axes may be disjoint from the set of \<output\> axes.

7. A region where \<input\> contains a single axis and \<output\> contains the same single axis is equivalent to the \<map\> mappings of avar1, except that in avar2, the delta may cause the value to cross over the default.

8. Allow axis tag as well as axis name.

Here follows a generic example of avar2 \<map\> where AX_0, AX_2 and AX_4 values are adjusted when the user designspace location is within the region specified as determined by its AX_0, AX_1 and AX_2 values:

```xml
  <map>
    <region>
      <input>
        <dimension tag="AX_0" minimum="700" xvalue="800" maximum="900"/>
        <dimension tag="AX_1" xvalue="150" />
        <dimension tag="AX_2" xvalue="75" />
      </input>
      <output>
        <dimension tag="AX_0" xvalue="750" />
        <dimension tag="AX_2" xvalue="140" />
        <dimension tag="AX_4" xvalue="140" />
      </output>
    </region>
  </map>
```


## avar2 use case examples

Here follow some suggestions for how to represent the distortion, parametric and HOI use-cases classes in .designspace. (Fences, the other proposed use case for avar2, are a special case of distortion.)

### Distortion use case

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

This example distorts the location (wght=700, wdth=150) to (wght=650, wdth=140), a 2-dimensional mapping not possible with avar1. Due to variations “tent” math, the effect is 1x at that location, reducing to 0x (no change) at the edges of the quadrant.

To determine delta values, the compiler normalizes input xvalue and output xvalue to 2.14 using the normal avar1 process, then subtracts the normalized input value from the normalized output value. If there an output for an axis but no input, the axis default value is used. Normalizing input and output xvalues gives:
* `wght` input: 700 → (700-400)/(900-400) = 0.6
* `wght` output: 650 → (650-400)/(900-400) = 0.5
* `wdth` input: 150 → (150-100)/(200-100) = 0.5
* `wdth` output: 140 → (140-100)/(200-100) = 0.4

Subtracting the normalized input value from the normalized output value gives:
* `wght`: 0.5 - 0.6 = -0.1
* `wdth`: 0.4 - 0.5 = -0.1

So for the region defined by `wght` [400, 700, 900], `wdth` [100, 150, 200] we have a delta set of [-0.1, -0.1] for the axes `wght` and `wdth`.

This region covers the whole “upper right quadrant” of the designspace, so we did not need to specify min and max. To be explicit and to demonstrate the use of min and max, we could have written the following to define exactly the same region and therefore the same action for the compiler:


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

### Parametric use case

This .designspace is for a parametric font with 3 user axes and 5 hidden parametric axes. We show a single mapping for the Weight axis acting on 3 parametric axes. Exactly the same method is used by the compiler as described above. The key difference with parametrics is that, since the output axis values are not specified in the input, the subtraction to determine the delta operates using the default values.

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

To calculate the deltas for compilation, we normalize input and output values:
* `XOPQ` input: 176 → (176-176)/(263-176) = 0.0
* `XOPQ` output: 260 → (260-176)/(263-176) = 0.966
* `XTRA` input: 562 → (562-562)/(640-562) = 0.0
* `XTRA` output: 370 → (370-562)/(324-562) = -0.807
* `YOPQ` input: 124 → (124-124)/(132-124) = 0.0
* `YOPQ` output: 132 → (132-124)/(132-124) = 0.966

Subtracting the normalized input value from the normalized output value gives:
* `XOPQ`: 0.966 - 0.0 = 0.966
* `XTRA`: -0.807 - 0.0 = -0.807
* `YOPQ`: 0.966 - 0.0 = 0.966

These values are multiplied by 16384 to become int16 values, stored as deltaSets in ItemVariationStore.

As with the calculation of outline variation deltas from master outline coordinates, we need to recall that deltas are based on accumulated values at the “corners”. This may be difficult to generalize given a) the deltas are ultimately in 2.14, and b) we want to allow arbitrary regions, not just orthogonal ones.

### HOI use case

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

### Adding avar2 tables to existing variable fonts

It will likely be a common occurence that we wish to add avar2 table to a compiled variable font. It seems natural to use identical .designspace mapping syntax in such cases, but this means a .designspace document without any UFO data. Perhaps `<source>` can optionally refer to a single compiled variable font, and `<axis>` elements can be omitted:
```xml
    <sources>
        <source filename="variable-font.ttf" />
    </sources>
    <axes>
		<!-- axis elements omitted, implied by the font -->
		<map>
			<!-- avar2 mappings here -->
		</map>
	</axes>
```
### Combining avar1 and avar2 tables in the same font

The avar2 specification allows for mappings for avar1 to coexist with mappings for avar2. The .designspace spec can support this: the avar1 maps are within each `<axis>` element; the avar2 maps are at the same level. For example:

```xml
  <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400">
      <map input="200" output="220"/> <!-- avar1 -->
      <map input="600" output="615"/> <!-- avar1 -->
      <map input="700" output="720"/> <!-- avar1 -->
  </axis>
  <axis tag="wdth" name="Width" minimum="50" maximum="200" default="100">
      <map input="70" output="80"/> <!-- avar1 -->
      <map input="150" output="160"/> <!-- avar1 -->
      <map input="180" output="190"/>  <!-- avar1 -->
  </axis>

  <map> <!-- avar2 -->
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
```

### Compiler optimizations
If an avar2 map is representable as avar1, the compiler should be free to use that option.


### Explicit deltas and normalized coordinates
We might consider:
* a relative mode, in case you want to work in deltas, with suggested representations within \<output\>\<dimension\>:
  * `mode="absolute"` (default), `mode="relative"` (deltas, optional)
  * `delta="..."` instead of `xvalue="..."`
* allow direct specification of normalized coordinates, `normalized="1"`


## Appendix: Comparison with other XML methods of representing variation math

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

```xml
    <VarStore Format="1">
      <Format value="1"/>

      <VarRegionList>

        <Region index="0">
          <VarRegionAxis index="0">
            <StartCoord value="-1.0"/>
            <PeakCoord value="-1.0"/>
            <EndCoord value="0.0"/>
          </VarRegionAxis>
          <VarRegionAxis index="1">
            <StartCoord value="0.0"/>
            <PeakCoord value="0.0"/>
            <EndCoord value="0.0"/>
          </VarRegionAxis>
        </Region>
      
	    <Region index="1">
          <VarRegionAxis index="0">
            <StartCoord value="0.0"/>
            <PeakCoord value="1.0"/>
            <EndCoord value="1.0"/>
          </VarRegionAxis>
          <VarRegionAxis index="1">
            <StartCoord value="0.0"/>
            <PeakCoord value="0.0"/>
            <EndCoord value="0.0"/>
          </VarRegionAxis>
        </Region>
	
		...

      </VarRegionList>
    
      <VarData index="0">
        <NumShorts value="0"/>
        <VarRegionIndex index="0" value="0"/>
        <VarRegionIndex index="1" value="1"/>
        <VarRegionIndex index="2" value="2"/>
        <VarRegionIndex index="3" value="3"/>
        <VarRegionIndex index="4" value="4"/>
        <VarRegionIndex index="5" value="5"/>
        <VarRegionIndex index="6" value="6"/>
        <VarRegionIndex index="7" value="7"/>
        <Item index="0" value="[-5, 12, 33, -23, -3, 1, -4, -4]"/>
        <Item index="1" value="[-3, 7, 19, -14, -1, 0, -2, -2]"/>
      </VarData>

    </VarStore>
```


### TTX CFF2 blend

CFF2 requires an ItemVariationStore, as well as delta sets defined with the `blend` operator:
```xml
	<BlueValues>
		<blend value="-15 2 -5 -1 -1 -2 2 5 5"/>
		<blend value="0 -2 5 1 1 2 -2 -5 -5"/>
		<blend value="670 6 -18 0 0 0 0 0 0"/>
		<blend value="685 -2 5 1 1 2 -2 -5 -5"/>
	</BlueValues>
```


### designspace \<source\>\<location\>

```xml
    <sources>
        <source filename="SourceSerif_c0.ufo">
            <location>
                <dimension name="weight" xvalue="0" />
                <dimension name="optical" xvalue="8" />
            </location>
        </source>
        <source filename="SourceSerif_c1.ufo">
            <location>
                <dimension name="weight" xvalue="394" />
                <dimension name="optical" xvalue="8" />
            </location>
        </source>
        <source filename="SourceSerif_c2.ufo">
            <location>
                <dimension name="weight" xvalue="1000" />
                <dimension name="optical" xvalue="8" />
            </location>
        </source>
        ...
    </sources>				


