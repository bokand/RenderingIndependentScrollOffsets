# Rendering-Independent Scroll Offsets

I propose to make scroll offsets independent from the media the page is being rendered on.

### Try it out
In Chrome 78.0.3886.0 and newer you can set chrome://flags#fractional-scroll-offsets to try out the proposal below.

### What does this mean?

Objects in a page can have sizes and positions in continuous values; the page can apply arbitrary transformations or positions using double-precision units. However, on a typical monitor, the final rendering occurs into a grid of discrete physical pixels. Pixels are indivisible units, so the rendering must be “snapped” to the nearest pixel.

In addition to “physical pixels”, the web is defined in terms of [CSS reference pixels](https://www.w3.org/TR/CSS21/syndata.html#length-units). These need not correspond to a physical pixel, although on a standard density monitor they typically do. Newer high density displays, typical on mobile phones, use UI scaling at the OS level to provide a higher quality image. In these cases, the reference pixel is larger than the physical pixels on the device. Many user agents also allow the user to change the size of the reference pixel (e.g. ctrl+/- in most browsers). The ratio of size between the reference pixel and a physical pixel is known as the [devicePixelRatio](https://drafts.csswg.org/cssom-view/#dom-window-devicepixelratio).

All major browsers today perform some kind of snapping to the scroll offset properties such as:

`window.scrollX`, `scrollY`, `scrollTo`

`Element.scrollLeft` and `scrollTop`

The implementations vary significantly (see interop section below); however, in all cases, these properties never reflect a granularity more detailed than physical pixels. In some cases, the granularity provided is less detailed than physical pixels.

### But they’re of type double!

The definitions of [window](https://drafts.csswg.org/cssom-view/#extensions-to-the-window-interface) and [Element](https://drafts.csswg.org/cssom-view/#extension-to-the-element-interface) both specify the scrolling properties as `double`. It may be surprising to learn that not all values of `double` are valid values.

While the type is double, the rendering engine will snap these values as it sees fit, in some cases, returning only whole integers. The spec is intentionally vague about when or how these values are snapped.

### What’s the problem?

The values that the engine clamps to are inconsistent and somewhat arbitrary. For example, some browsers will account for ctrl+/- zooming to allow more precision, but not pinch-zoom or CSS transforms. Furthermore, the behaviors vary significantly between major browsers.

One could argue these are bugs and should be fixed to provide specific, interoperable behavior. However, I’d like to argue that clamping scroll offset values at all can produce surprising and unintuitive behavior and we should change to a model where offsets are unclamped.

Consider a device with an unusual UI scaling factor. The Nexus 5X has a devicePixelRatio of 2.625. This means that one CSS reference pixel is 2.625 device pixels. Suppose the author wishes to scroll the page:

```
window.scrollTo(0, 20);
```

Seems simple enough. What do we expect `window.scrollY` to return after running the above snippet? You might be surprised:

```
console.log(window.scrollY);
// 19.809...
```

Why is this? When we request scrolling the Y coordinate to 20 CSS reference pixels, the browser must render the translation at a physical pixel boundary. `20 * 2.625 == 52.5` physical pixels. Since we cannot render half a physical pixel, it must be clamped to an integer, in this case, it is truncated to 52. When the page requests `window.scrollY`, we must return the value in CSS reference pixels. Because Chromium truncates the offset itself, and not just the rendered location, the result is `52 / 2.625 == ~19.8`.

This can be very surprising for authors. You’d expect:

```
element.scrollTop = value;
element.scrollTop === value
```

to hold. Not only does it not, it makes life difficult for developers because the behavior will be different depending on the device. Script that works well one a development device might break on a user’s device.

One could argue that we’re returning the rendered position so, while surprising, it’s a reasonable thing to do. However, which offsets are valid can change even on the same device. Consider a page loaded on a standard density monitor, at 200% zoom. In this case, scroll offsets can be changed at increments of 0.5. However, if the user zooms back out to 100%, only whole integers are allowed. What are we to do with the now invalid offsets? If we truncate them, should we fire scroll events? Consider a pinch-zoom which continuously changes the size of the CSS reference pixel. Changing the properties of the rendering media should not conceptually cause scrolling.

Here are some real-world examples from Chromium’s bug tracker of authors finding surprising bugs/behavior:
 * https://crbug.com/873870: Scroll sequence sometimes ends with tiny backward scroll
 * https://crbug.com/890345: Can't set scrollTop value precisely if zoom setting is not 100%
 * https://crbug.com/931455: Layout bug when using fractional width or height

### Proposal

The scroll properties of the `window` and `Element` objects should be made independent of the media they’re being rendered into. Any valid double-precision number should be a valid offset and pixel-snapping is purely an artifact of rendering.

This means the value set from script is the value that’s stored and returned during a read. Pixel snapping still occurs when rendering the page, but it doesn’t affect the scroll position which may fall between physical pixels.

This should be much more intuitive to authors and provide fewer surprises. It also means devicePixelRatio-changing operations don’t have side-effects on scroll offsets like they do today.

### Interop Analysis

Tested on highDPI(devicePixelRatio == 2):
 * Safari: 12.1.1 (14607.2.6.1.1)
 * Chrome: 78.0.3880.4 
 * Firefox: 70.0a1 (2019-08-15) (64-bit)
 * Edge 44.18956.1000.0

##### User Scrolls (i.e. trackpad scrolling)
All browsers allow smooth scrolling on high-DPI devices (i.e. the scrolls really are happening at a physical pixel granularity), however:

* Chrome/Firefox: User scrolls that land on physical pixels are reflected in `scrollY`. I.e. A trackpad of 100.5 pixels will cause `scrollY == 100.5`. Changing zoom level affects these values (zooming in provides finer precision).  
* Edge/Safari: `scrollY` always truncated to integer

##### Programmatic scrolls to physical pixels
`window.scrollTo(0, 100.5)`
* Chrome: Untruncated  
* Firefox/Edge/Safari: Truncated to 100

##### Programmatic scrolls to sub-physical pixels
`window.scrollTo(0, 100.678)`
* Chrome: Truncated to 100.5  
* Firefox/Edge/Safari: Truncated to 100

##### Zooming
Zoom in, scroll to sub-pixel offset, zoom out and back into the original level.
* Chrome/Safari: Offsets can visibly change - appears they’re really stored as physical pixels
* Edge/Firefox: Offsets are unchanged once original zoom is restored - appears they’re really stored as CSS pixels
  * Firefox: Has an interesting property that once a scroll offset is set, e.g. 100.333, zooming out to a level where that would be invalid doesn't change this value.

All browsers fired ‘scroll’ events during zooming, even if the scrollY value didn’t change.
