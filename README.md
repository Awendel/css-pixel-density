# css-pixel-density
Proposal for a new CSS property "pixel-density".

Link to CSS Drafts issue: https://github.com/w3c/csswg-drafts/issues/7848

## Abstract
Offer a way for a given layer to adjust its internal resolution by a multiplying factor. The default should be "1" or "100%", which is the resolution that is currently chosen. 

If I set the value to 0.5 or 50% it should half the internal resolution of the layer (and hence put less pressure on the rasterizer hardware and gpu memory etc).

```
.layer{
  pixel-density : 0.5; /* halfs the resolution */
}

.zoomIn{
  pixel-density : 200%; /* doubles the resolution in preparation for zoom in */
  will-change : transform;
}
```

## Background
Every year devices become higher and higher resolution (especially smartphones, most have very high dpi displays).
At the same time the scope and scale of what web apps can render (think Figma, data visualisation, ...) becomes larger and larger.
Moreover, the amount web users increases every year with largest growth coming from developing countries where sadly the majority have access to only low end hardware.

## Utility
- make graphics intensive animation run smooth (60fps) even on low end devices, by temporarily lowering the resolution for the duration of the animation
- offer way for low end devices to use graphics intensive application with useable framerates
( similar to how common it is to lower the resolution in gaming, especially on 4K displays etc)

## Compatibility & Easy of Integration
The proposed property would integrate very well with the following existing properties:

- **transform**, esp. scale
- **will-change**, especially when set to **"transform"**. Will-change "locks" the internal resolution of a layer already, but it doesn't provide a way to lower it, especialling when "zooming" out. It is currently not enough to ensure every animation runs smooth. I will attach some test cases to prove this and show the only way to run it smooth would be to lower the resolution.
Moreover, it would be very simple to integrate into the existing render logic of browsers. Since there already is a mechanism to calculate the layer resolution (multiplying screen resolution by devicePixelRation and transform:scale), it would be just another multiplier in that computation.

## Lack of alternatives
The only way to achieve this currently is by pushing all render logic to canvas, where one can control the resolution.
Sadly this has led to a situation where all graphics intensive application use a custom WebGL render stack (Figma, Google Maps etc),
even though all rendering could be done with plain HTML.
Plain HTML is also by nature more accessible and eases the barrier to entry for new developers.

## Reasons why it should be added to CSS standard
- inline with new Houdini approach of exposing low level primitives to web developers which can be combined to greatly customise render experience
- simplicity + intuitiveness of API
- simplicity of integration into existing browser rendering pipelines (devicePixelRatio etc)
- addresses 2 main concerns for web developers who do animations:
(1) animations not running smooth (fix via lowering / downSampling resolution)
(2) animation not being sharp (fix via increasing / upSampling resolution)
- making web more accessible by allowing lower end devices to render graphics intensive content
- making web more accessible because now plain HTML can be used without having to implement this in custom WebGL / Canvas rendering stack
- has potential to reduce loading time of HTML pages when a lot of content is rendered, by lowering the resolution before content is rendered, hence taking pressure of UI thread, it will appear faster on the screen, then one could increase resolution async in order to sharpen it again but not blocking anything

## Edge Cases

### Combination with transform : scale
**pixel-density** is not an absolute property. Instead it scales relative to what the browser determines to be the correct rasterisation scale for a given layer.
Hence values won't have to be manually calculated when using transform:scale (e.g. multiplying), but instead can still be used relatively

## Combination with will-change: transform
will-change transform is meant to "lock in" a given rasterisation scale based on when the property was set.

This is currently not respected by all browsers. Only Chrome does so reliably. Ideally we'd get all browser vendors to treat will-change: transform as a "rasterization-lock". Whenn adding **pixel-density** we're able to modify that behaviour in 2 different ways:
- scale the raster resolution when setting will-change:transform, either increasing it (e..g zoom in expected) or decreasing it (zoom out expected)
- change the rasterization scale during an ongoing will-change transform animation. By changing **pixel-density** we force a recalculation of the raster resolution, even if it has been previously "locked in" via will-change transform. This might be neccessary, when zooming out by a lot, which could lead to gpu memory overflow, if the resolution is not adjusted in time

## Nested pixel-density
There is 2 possible approaches to this:
- multiply nested values, similar how it's done with transform:scale
- treat each value as independent of its possible parent values. This is the preferrable approach, since it leads to less ambiguity and fewer calculations. It is rare that a nested value would be neccessary, but they should be treated relatively to what the browser would have painted them, regardless of if a parent has a different value applied to it.

## Examples


### Decrease page load time
```
const renderTarget = document.querySelector("#page")

// substantially lower resolution to decrease GPU memory pressure and speed up shaders
renderTarget.style.pixelDensity = '0.25'

// perform expensive initial rendering, especially in a SPA

window.addEventListener("load", function(ev){

  // reset resolution to native on of screen so content becomes sharp
  // this should be quite fast and not add much block to the UI thread
  // since layout calculations have already been performed and don't need to be redone
  
  renderTarget.style.pixelDensity = '1'  

})


```
