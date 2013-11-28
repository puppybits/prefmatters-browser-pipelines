# Journey down the browser stack

--- 

## Browser are built to render STATIC pages

---

### Simple Browser Pipeline

* **Database/REST** - magic is sitting on the backend
* **Network** - download HTML and resource
* **Scripting** - run JavaScript 
* **Reflows/Layout** - calculate the positions on DOM elements
* **Rendering** - draw to the screen 
* **Idle** - do nothing until the user clicks a hyperlink

---


# Journeyman’s tip #1: 
## 60% - 80% of the performance impact to users of a single web app and static pages is on the network and back-end. 

Check HAR files, REST API performance and database query performance before mucking with the browser. Front-end browser issues are things like scroll performance, long page loads (because of the render cycle), janky framerates.

---

## The JavaScript wrench in the browser stack

---

## 1. Reflows

---

## Reflows

Reflows or recalculate styles is the browser needing to traverse the DOM tree and CSS rules to figure out the right positions for each element. This is not part of drawing to the screen. 

---

## JavaScript methods can trigger browser sub-systems to run IMMEDIATELY!!

Some functions will pause JS execution and run a sub-system in the browser before moving to the next line.  

Generally these are DOM (or jQuery) methods.  

    offsetLeft, offsetTop, offsetWidth, offsetHeight clientLeft, clientTop, clientWidth, clientHeight scrollLeft, scrollTop, scrollWith, scrollHeight, scrollX, scrollY, scrollBy(), scrollTo() document.width, document.height,  image.width, image.height getClientRects(), getBoundingClientRect(),  getPropertyCSSValue(), innerText


###### NOTE: this list is not a complete list and will change as new browsers versions are released. 

---

## CSS consequences

* not every CSS method has the same performance. There are some methods that cause more time in the CPU or GPU or in the layout engine.
* **don't pre-optimize** just make better decisions up front. Understand the cost and if there are performance issues the users/designs see on the low-end devices then you should know where to check for bottlenecks first
.

    border-radius, box-shadow, text-shadow, image-mask, filters, opacity, transform, visbility: hidden, gradients, font-family(web fonts), top, left, float, display: table, display: table-cell

---

## Partial Reflows

* reflows don't have to start at the document root and trigger down. Sometimes they will start at a sub node and trigger down. Sometimes they can walk up to higher nodes in the DOM tree then spiral down.
* stay far, far away from tables. even if something looks like a table you better have a damn good reason to use the `table` tag. Setting `display: table` or `display:table-cell` is the same thing. If you MUST vertically centered content use `display: flexbox` (the new flexbox) and polyfil for old browsers.
* `float` will also take more time in the reflow engine. Lots of floats next to each other, changing one will reflow them all.


---

## Layout Boundries - opting out of reflows

* you can decide when to set hard boundries that the reflow engine will never cross. 
* `position: absolute` is a good first step but would allows stop the reflow beast.
* Use `position: absolute + display: block + height: XXpx + width: XXpx + overflow: hidden | scroll` to force the reflows to stop before they reach the document
* Use Boundarizr to see layout boundries
* go sparingly on hard coding the layout boundries. You are implictly opting out out of the browser from doing it's job, which is to layout out a bunch of elements all together. 

---

## Layout Boundries - opting out of reflows (continued)

##### Journeyman's Tip: Don't waste your time with setting layout boundries unless you first stop the layout engine from over running. Stopping the layout engine from running too often will generally give you much better improvements. Also because you're opting out of the normal reflow process, DON'T ever talk about layout boundries until you've inspected, found all the other bottlenecks and fixed those first and need a little bit more.

---

## 2. Memory Allocations

---

## Memory 101

* even through we don't write in C, you still need to understand how memory works and what are the expensive operations.
* understand the heap and call stack
* heap allocations (finding free memory addresses in RAM) is an expensive operation, 
* heap deallocations and reallocations are horribly ineffectant. Reuse memory when you have performance concerns. This will save massively amounts of RAM and CPU processing time.
* look for frameworks that support data binding or object mutation observers. More often than not they have made the smart choice upfront that saves the CPU/RAM a ton plus requires less boilerplate code from the developer.


---

## Memory 101 (continued)

* the Garbage Collection cycle automatilly picks up items and frees memory. JS doesn't have any control of when the GC will run. The more object, more pointers and larger the heirachies the longer it will take to complete the GC cycle.

##### Journeyman's Tip: Memory is more important than every else. JavaScript is not the first place to look for memory issues. Check the images, network calls, poor DOM use before looking at JavaScript functions. Never worry about memory leaks. The Garbage Collector will get most of your memory. Any unremove listens on DOM objects won't really matter much unless you're creating too much DOM crap.

---

## Images - the massive beast you never saw

* PNG, JPG, GIF are image _compression_ formats. Meaning the size you see in the file browser is NOT the size in RAM or that needs to be processed in order to get the information to the screen. Everyone gets converted back to a format similar to BMP in order to be shown on the screen.
* A general rule of thumb is PNGs are 3 time larger in RAM than on disk.
* Animated GIFs cause a consistant performance hit. `visibilty: hidden` does remove the element from the layout engine so it will still encore CPU/GPU hits. `display: none` will reomve it from the layout engine but still keeps the image in RAM. 
* Remove all images expecially GIFs from the document as soon as you don't need them. 

---

## Images - the massive beast you never saw

* SVGs can be just as heavy. They need to be processed in the CPU and turned into an image. They might need to be reinflated each time they are drawn. I had this issue once on a canvas cloth simulation. The SVG was getting reprocessed each time the cloth simulation was updating instead of just using the BMP output. Switching to a PNG tripled the framerate.
* `&lt;img src='some.img'&gt;` and `image.src = someUrl` is a MASSIVE task. It opens a network connection, downloads a large amount of data, decompresses the image, calculates the reflows, moves the entire image to RAM (usually GPU RAM) and then has to repaint the screen.

##### Journeyman's Tip: Images and setting image src one of the most massive things you can do. They dwarf the CPU and memory that most of you JavaScript can crack through.

---

## Template over use

* templates are great and easy to use but they cause a lot of memory to be created
* Most importantly, **KEEP ALL BUSINESS LOGIC OUT OF THE TEMPLATES!!!**
* Let me repeat that again! **KEEP ALL BUSINESS LOGIC OUT OF THE TEMPLATES!!!**
* when you remove things from the DOM and readding DOM you're digging a hole and filling back up again. There's NO reason to do this.

##### Jouneryman's Tip: Some frameworks tie a change in data to an instant event notification which, by default, triggers dellocating a portion of the dom, recreating the SAME dom with one minor change and placing it on the document. This is crazy, needless wastefullness. We had a case where a page went from 3 second load time to 35 seconds on desktop chrome because of this poor pattern.

---

## 3. Rendering pipeline

---

## Rending overview

* This happens after the reflow is complete. Also this is a gross oversimplifcation and will change a lot in newer browser versions. But if you know what the browser is doing in the render pipeline you know where to go to look for bottlenecks.
* **opacity compositing.** Flatten down anything that has transparent regions into one image.
* break up the page into subregions. (Generally in a multiple of 64 pixels because of the GPU hardware pipeline architecture). This is how modern browsers are tuned for better scrolling performance, especially on mobile.
* each subregion is created as a texture and sent to the GPU. The texture is stored on the GPU RAM. Doing functions like moving, scaling and rotating are hyper optimized on the GPU.

---

## Rending overview (continued)

* **layer compositing.** Some elements are broken off into their own GPU layer. `position: fixed` or `transform: something-that-ends-in-Z` or `z-index: XX` will cause GPU layers to be created. Each browser will handle this very differently. `z-index` is the least likley to be created as a GPU layer.
* **bliting.** After the layers on composited down into one image that image is blited to the screen. Generally the GPU takes care of all of this.


---

## Software compositing

* remeamber that EVERYTHING is a square. any css effects or images or anything see-through needs to have the pixel color calculated and blended down to one color.
* the biggest affector here are usually images with **alpha channels**.
* some **CSS effects** can have greater impact during the software compositing cycle. `image-mask`, `filter`, `text-shadow`, `box-shadow`, `border-radius` are all big performance skinholes in this cycle.


---

## Software compositing (continued)

* pre-caching. When issues happen you can pre-cache these. Alphaed images can remove the transparency, CSS effects can be made in photoshop as static, non-transparent images, they can be removed with javascript during a heavy render cycle. Know you're options but don't do any work until you have a valid reason to.

##### Journeyman's Tip: I am NOT saying don't use transparent images or css effect! I'm say use them to you heart's content. It's easy to code and quick to adjust. But IF..IF..IF you have rendering issues look to pre-cache them. 

---

## Browser scrolling optimizations

* every scroll requires a repaint of the entire view. Breaking the page into GPU tiles makes it super easy for the GPU/CPU. Matrix operations are hyper optimized in the GPU.
* If a DOM node that is partial displayed in side the GPU layer is visually modified, then that GPU tile needs to be redraw (aka repainted). Safari 6.1 does an awesome job of showing this on the desktop and on a connected iOS device.


---

## Browser scrolling optimizations (continued)

* In Chrome see were the GPU tiles and GPU layers are by going to Dev tools > General > Rendering > "Show composited layer border". GPU tiles have blue lines. GPU layers are orange.
* In safari go to web inspector > resources (on the left) > the 3d layer stacks on the upper right of the HTML source code.
* Safari will have a number on the upper left fo the GPU tile/layer. This tells you how many times it's been repainted. 


---

## Hardware Compositing

* GPU layers will be composited as the last step. They need to re-calcuate the opacities overlaps and stacking order.
* The most relaiable way to force a GPU layer currently is setting a transform with a z property, like `transform: translateZ(0)`.
* Safari 6.1, if you click the DOM node that is a GPU layer it will tell you the memory of each image in the GPU's RAM.
* Safari 6.1, click on the layer name in the layer inspection panel to see WHY it was turned into a GPU layer


---

## Hardware Compositing (continued)

* GPU Layers are not in the normal JS garbage collection. It's in the GPU RAM. The browser will trigger when to clean house on the GPU layer.
* poor use or over use of GPU layers will kill your perfromance like non-other. Don't much with this stuff unless you know what you're doing. The GPU is not fast at compositing the opacties of multiple images. It's better to have the largest static image possible and then let the GPU move it around on the screen.

##### Journeyman's Tip: Safari 6.1's GPU layer inspection tools are by far the best of the current browsers. Skip Chrome and go straight to Safari to look at GPU memory, layer compositing, repaint cycles. IE 11 show the time spent on decompressing images, garbage collection time and network calls. Chrome is still the go to for general all-in-one working.

---

# Make better decisions upfront

---

## Browsers do tons of amazing optimizations already

* browsers are highly tuned. It's for a reason. They are GREAT at what they do. They are very flexible. You can easily abuse the browser. 
* **DON'T THINK YOU'RE SMARTER THAN THE BROWSER** Most of the time they do the right thing. Lots of JavaScript can leave the browser guessing at what's next. 

##### Journeyman's Tip: Understand when you could be causing heavy loads on the browser. If you have a simple way to not hurt the browser then don't hurt the browser. If you don't have a simple solution then don't worry about it. Most likely the browser will be just fine. Only optimize AFTER you have found cause for a bottleneck.

---

## Do the right thing first

* Now that you know all of these important parts of the browser pipeline, **PICK A GOOD FRAMEWORK!!**
* Make better descions when coding. If 2 things are easy and you think there might be problems, then do the more performant thing.
* **NEVER PRE-OPTIMIZE**. Don't do that harder thing just because it's more performant. Only waste the time when you have a damn good reason that the performance sucks.
* **NEVER OVER-ARCHITECT**. Don't make things too abstract or use complex patterns or build something more than what you need for today. Tomorrow if you need to add more features on something then do that tomorrow but only if there is no other options.

---

## Lazy loading

* don't load too much. Load what you need and prioritize loading assets that the user will notice. like loading image just before the user scrolls up.

---

## Batching network requests

* wireless and 3G radios take a lot of power to boot and keep on. Do as much work on the network all at once and let it power down. Better for battery life and better to have resources locally.

---

## Lazy rending

* don't render all the DOM elements as once. Spread the load over multiple run loops.
* watch out of the memory creeping up. Use object pooling to have less creep and a max limit on memory.

---

## Defered and Debouce

* a poor man's hack for some huge waste of the system. 
* I've debouces a template destuction/reconstuction render cycle to control a spamming of trigger events.
* **defering for for the next run loop and debouce are NOT fixes.** They are nasty, nasty hacks. Don't "architect" them into your code and tell me it's good. They are not. It's a hack for abuses from other classes or your framework.
* Build better the first time so you don't need to debounce or defer work.
* If you're JS framework normal patterns have you deboucing things something is majorly wrong with your framework. 

---

## Object Pooling 

* NEVER let the GC touch you. 
* Super fast GC run times so it never hits your performance. DOM pooling gives the biggest wins. Pooling on JS arrays or models you have is generally useless. Most of the work isn't in JS its in the browser somewhere.
* keep all objects in an array and reuse them.

---


# Case Study - PerfView

[https://github.com/puppybits/BackboneJS-PerfView](https://github.com/puppybits/BackboneJS-PerfView)

---

## Making better choices up front

1. Let the browser do it's normal processing as much as possible. Don't constantly use javascript to change things for the browser. Batch changes all at once to minimize the impact to the default browser pipelines as much as possible. 
2. Object pooling. Don't draw too many dom nodes. Only so much can be on the screen at once. Only create a few and reuse them. This keeps the memory from growing and shrinking. 
3. Don't rerender templates. The framework’s default process is to destroy and recreate DOM every time the model changes. Keep all the business logic out of the template. Pre-render and set as display: none for any elements that aren't a big load. 
4. Layout boundaries. Set each row so they the reflows are internal to a given row but don't effect the entire document. 

---

## Frames per second and basic memory usage

5. Look for bottlenecks. 
6. Inspect memory performance of the dom nodes using the timeline.

![fps](../Chrome-DOM-Node-Count.png)

---

## Heavy CPU usage

7. Timeline still is slow. Run profiler. Everything is under the update function. Break out processes into discrete functions. Add names to anonyoums functions. Run profiler. Percentage of time in calculateHeight is crazy high. Add in option to allow developer to set a static height. Speed gets much faster.

![profiler](../Perf_Append_1.Calc_Height_Percent.png)

---

## Memory bottlenecks

8. Scrolling lots chugs. Open IE 11 dev tools. Look at memory and network loads when scrolling and loading new images. Dynamically disable image updates when the scrolling is too fast, re-enable when stopped. 

---

<img style="width: 40%" src="../IE11-append-on-document.png">

---

## Going Mobile

9. Run on iOS. Nothing works. iOS stops all JavaScript from running with the body is scrolling. Add in iscroll framework.
10. Everything works but the scrolling is crazy fast and JS can't keep up. Slow down iscroll to a reasonable speed for JS to process. 

![mobile](../Perf_iPad_1.Dynamic_Heights_Fast_Scrolling_FPS_DocumentFragment.png)

---

## Random crashes of mobile safari

11. Scrolling too much crash mobile safari. Open up Safari 6.1. Inspect memory profile is fine. GPU layers climb crazy high. After scrolling stops for 4 or 5 seconds the memory usage drops back down to 5 MB. Readjust GPU layers for the scroller, body and reuse views. GPU memory usage never spikes high again.

![layers](../Safari-layers-panel.png)

---

## Slow-downs with Timeline or Profiling on

12. FPS is degrading a lot when timeline is turned on. Write a super light FPS tracker. Run with the timeline off and FPS reports much closer to what it looks like. Less slow down.

![frame-rate](../Chrome-DOM-Node-Count-FPS.png)

---

## Going for extra credit?

13. Mobile safari runs from 60 to 45 FPS. Can it get faster? Maybe reflows can be brought down more. Force only 2 reflows ever instead of a bunch of micro-reflows. Move all items to a document fragment before editing. CSS dymanic heights are broken now. Document fragments can't calculate heights on a fragment. Set static heights. Performance still doesn't show a sizeable jump. Degrades on IE 11. Scrap it.

---

## Getting stupid

14. Max run interval is spiking when nothing happens. Time for micro-enhancements that to absolutely nothing for the user, aka leaks. Re-write FPS counter to use new Ecmascript Typed Arrays (with pre-filled length normal arrays as a fallback). Nothing every gets collected by the GC anymore. This did NOTHING for the user. Unless you're building a framework that's going be to use a lot this would have been POINTLESS!!

---

## Journeyman's Checklist

* pick a good framework.
* understand the browser pipeline.
* make better decsions up front. Don't make the browser overwork. Don't pre-optimize.
* Stay in the normal browser pipeline as much as possible.
* Look for bottlenecks where the most of the work is done. Use profiling tools to have clear facts about what work is being done, how much and how it compares to other parts of the system.
* Only fix things the user sees. Don't over-optimizel you start spending lots of time and make little or no sizable impact on user performance.
* The best performance optimization is the one you don't have to do. Have UX change the design to be more memory effeciant. Favor pagination over scroll view. 

---

## Appendix

Webkit Performance - WWDC 2013 - {Simon Fraser}  
[Wilson Page, Layout Boundaries - http://wilsonpage.co.uk/introducing-layout-boundaries/](http://wilsonpage.co.uk/introducing-layout-boundaries/)  
[Paul Lewis, Boundarizr - https://github.com/paullewis/Boundarizr/](https://github.com/paullewis/Boundarizr/)  

---

