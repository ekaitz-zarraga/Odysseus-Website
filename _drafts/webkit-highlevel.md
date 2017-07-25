WebKit offers many interesting blog posts about how their software works. However I am left unsatisfied in that they do not offer a post pulling it all together with a few extra tidbits I discovered by reading their codebase myself. So I thought I might enjoy writing that post myself. 

### Network, parsing, DOM

When a page is visited for whatever reason WebKit (which runs in a seperate process from Odysseus) tells Odysseus's process that it needs that page. This "UI process" then requests the page using [libsoup](https://valadoc.org/libsoup-2.4/index.htm) (in the GTK port) and coordinates the request lifecycle with the the "WebCore process". There is surely more interesting aspects of this networking component (which includes caching), but we'll ignore it to keep this highlevel.

As the data comes in it gets parsed in relatively straightforward and handwritten code. When it comes to HTML the AST it gets parsed into is called the DOM, which upon construction is given a chance to alter the browser environment which may involve:

* Immediately requesting additional resources, which it can use for some resource.
* Tell the UI process to change the tab's title, favicon, etc.
* Register event handlers on itself to be called upon user interaction.
* Substitute in an alternate DOM hierarchy for display.

The loading of resources elements may go though a different network stack with checks on whether the site and/or an installed content blocker wants to prevent or alter the load. 

### Applying CSS rules

Once we have a AST for HTML and CSS, we need to combine them to make a "style tree". And making this fast is difficult, and there are several techniques webkit uses.

First they use indexes over selectors matching certain classes, IDs, and tagnames to find an initial set of selectors for consideration. This makes simple selectors very fast, but to make path selectors fast a technique used bloom filters are used. A bloom filter is a datastructure which can be used to determine very quickly whether a CSS rule probably matches or definitely doesn't. Finally the remaining selectors are lazily compiled into machine code to double check it very quickly. 

Having found those CSS rules, CSS cascade needs to apply. Constructing an array of CSS values, and assigning to certain indices for each property in each rule. As a final step some code generated by a Perl script is used to apply CSS animations and restructure the data into space efficient structures. 

### Layout and Rendering

From the style tree's display property, as well as aspects of the DOM, it is trivial to construct a "rendering tree". A few traversals over this and WebKit knows where to put everything onscreen. After another traversal a "tile tree" is generated with bitmap images to draw onscreen. The tile tree is then handed to the window manager or OpenGL for it to composite those images. 

### Hittesting

Whenever mouse input comes in the render tree is hittested to figure out which element was operated upon, through which registered event handlers from the DOM tree are triggered. These event handlers may in turn either perform simple DOM manipulations themselves or enqueue an edit event on the undo/redo stack and let it manipulate the DOM. 

Interestingly form inputs call into the OS just for native form rendering, and handles it's input through the DOM layer. Also the MathML and SVG DOMs use much the same architecture, even though most their logic is in rendering and leaves the DOM implementation rather bare. 

### Note on JavaScript

The way WebKit's multitier JavaScript engine works, as well as it's garbage collector is well documented on the WebKit blog. However it is worth noting that the APIs exposed to JavaScript mostly calls into one of these components or one of the underlying libraries it uses from the OS. However there are a few APIs that do things WebKit doesn't already do, and some of those like GeoLocation can be extremely useful. 
