There are 

In principle an app could embed a relationship between user axes and parametric axes. 

This process is known as “blending”, and allows the font to be controlled either by user axes or by parametric axes. It comes at a significant cost in extra data.

In one test font, using avar2 to map user axes to parametric axes (effecitvely the “blending” happens live) enabled a size reduction from 1.7 MB to 330 kB (~80% compression). The avar2 table itself required around 3 kB.

Parametric fonts, explored by Donald Knuth, David Berlow and others, offer fine-tuned control of specific aspects of their design. Such aspects typically include the thickness of the vertical strokes, thickness of the horizontal strokes, the overall width of the font (based on a key glyph), and direct control over vertical zones including cap-height, ascender height, x-height and descender height. In an OpenType variable font these parameters can be encoded as design-variation axes, as has been successfully demonstrated by Type Network with the fonts Amstelvar, Roboto Flex and others.



 in recent years by Type Network 






Just as points in a glyph can have a delta applied, so can axes themselves have deltas applied to them. So let’s consider the inputs and outputs. The inputs of a variable font are the axis settings. THese get normalized and hten optionally transformed by avar to get new nornalized axis settings. IN that, avar2 is no different from avar1: we have some inputs and the outputs are in general different.

Instances arrived at in fonts with an avar2 table can in general be arrived at by the font without the avar2 table.

In practice the difference is easier control by a user or system:

* fewer axes need to be set
* axes have more intuitive values: the weight=700, width=50 reflects the designer’s intention for a Bold Narrow font, and the instance can be correctly referred to.





Example, assuming a `wght` axis with minimum 300, default 400 and maxmium 700, an `avar` table that has v1 mappings for `wght` of -1 to -1, 0 to 0, 0.5 to 0.4, 1 to 1, and with other axes:

1. User value of 625 converts to (625-400)/(700-400) = 0.75
2. `avar` version 1 maps 0.75 to (0.4 + (0.75-0.5) / (1.0-0.5) * (1.0-0.4) = 0.7
3. `avar` version 2 maps 0.7 adds a delta determined by the final normalized values of 




