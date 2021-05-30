[![Build Status](https://travis-ci.com/crystallabs/crysterm.svg?branch=master)](https://travis-ci.com/crystallabs/crysterm)
[![Version](https://img.shields.io/github/tag/crystallabs/crysterm.svg?maxAge=360)](https://github.com/crystallabs/crysterm/releases/latest)
[![License](https://img.shields.io/github/license/crystallabs/crysterm.svg)](https://github.com/crystallabs/crysterm/blob/master/LICENSE)

# Crysterm

Crysterm is a console/terminal toolkit for Crystal.

It tries to follow closely the implementation and behavior of the libraries that inspired it,
[Blessed](https://github.com/chjj/blessed) and [Blessed-contrib](https://github.com/yaronn/blessed-contrib)
for Node.js. However, being implemented in Crystal (a proper OO language), it tries to use the language's
best practices, avoid bugs and problems found in Blessed, and also (especially in the future) incorporate
some aspects of [Qt](https://doc.qt.io/).

## Trying it out

```
git clone https://github.com/crystallabs/crysterm
cd crysterm
shards --ignore-crystal-version

crystal examples/hello.cr
crystal examples/hello2.cr
crystal examples/tech-demo.cr
```

(If you get an Exception trying to run the "tech-demo" example, maximize your terminal window and try
again.)

## Screenshots

Animated demo

![Crysterm Demo Video](https://raw.githubusercontent.com/docelic/crysterm/master/screenshots/2020-01-29-1.gif)

Layout engine (masonry layout)

![Crysterm Masonry Layout](https://raw.githubusercontent.com/docelic/crysterm/master/screenshots/layout.png)

Transparency, color blending

![Crysterm Color Blending](https://raw.githubusercontent.com/docelic/crysterm/master/screenshots/transparency.png)

## Development

### Introduction

As mentioned, Crysterm is inspired by Blessed, Blessed-contrib, and Qt.

Blessed is a large, self-contained framework, whose authors have, apart from Blessed itself, implemented all the
necessary/prerequisite components themselves, including an event model (a copy of an early Node.js EventEmitter),
complete termcap/terminfo system (parsing, compilation, output and input from terminal devices, i.e. a better
version of Ncurses), all types of mouse support, Unicode handling, color manipulation routines, etc.
These implementations have been embedded in the source code of Blessed itself, reducing the potential for their
reuse.

In Crysterm, the equivalents of those components have been extracted into individual shards to make them available
to the whole Crystal ecosystem. The event model has been implemented in
[EventHandler](https://github.com/crystallabs/event_handler), color routines in
[term_colors](https://github.com/crystallabs/term_colors), terminal output in
[tput.cr](https://github.com/crystallabs/tput.cr), and terminfo library in
[unibilium.cr](https://github.com/crystallabs/unibilium.cr).

Unibilium.cr represents Crystal's bindings for
a C terminfo library called [unibilium](https://github.com/neovim/unibilium/), now maintained by Neovim.
The package exists for a good number of operating systems and distributions, and one only needs the binary
library installed, not headers.
There is also a mostly working Crystal-native terminfo library available in
[terminfo.cr](https://github.com/docelic/terminfo.cr) but, due to other priorities, trying to use that instead
of unibilium is not planned. Both unibilium and native terminfo implementation for Crystal were implemented by 
Benoit de Chezelles (@bew).)

Crysterm closely follows Blessed, and copies of Blessed's comments have been included in Crysterm's sources for
easier correlation and search between code, files and features. A copy of Blessed's repository also exists in
[docelic/blessed-clean](https://github.com/docelic/blessed-clean). It is a temporary repository in which
files are deleted after their contents are reviewed and discarded or implemented in Crysterm.

High-level development plan for Crysterm looks as follows:

1. Improving Crysterm itself (fixing bugs, replacing strings with better data types (enums, classes, etc.), and doing new development).
1. Porting everything of value remaining in blessed-clean (most notably, parsing of terminfo command responses from terminal isn't implemented, mouse support is missing, and a number of final widgets)
1. Since Blessed repository is no longer in active development, reviewing the updates Blessed has received in forked repositories neo-blessed and blessed-ng, and using them in Crysterm where applicable
1. Porting over widgets & ideas from blessed-contrib
1. Developing more line-oriented features. Currently Crysterm is suited for full-screen app development. It would be great if line-based features were added, and if then various small line-based utilities that exist as shards/apps for Crystal would be ported to become Crysterm's line- or window-based widgets
1. Adding features and principles from Qt

For other/more specific development and contribution ideas, scan sources for "TODO", "NOTE", or "XXX",  see file `TODO`, or see general Crystal wishlist in file `CRYSTAL-WISHLIST`.

### Event model

Event model is at the very core of the Crysterm library, implemented via [EventHandler](https://github.com/crystallabs/event_handler).

The events used by Crysterm and its widgets are defined in `src/events.cr`.

### Class Hierarchy

1. Top-level class is `Screen`. It represents a physical device / terminal used for `@input` and `@output` (Blessed calls this `Program`)
1. Each screen can have one or more `Window`s (Blessed calls this `Screen`). Windows are full-screen and always occupy full screen
1. Each window can have one or more `Widget`s which implement the final usefulness

Widgets can be added to windows directly, but some widgets are particularly suitable for containing child elements.
Most notably the `Layout` widget which can auto-size and auto-position contained widgets in the form of a grid or inline (masonry-like) layout (`LayoutType::{Grid,Inline}`).

There is currently no widget that would represent a GUI-like window, like `QWindow` or `QMainWindow` in Qt.
I am considering adding such a widget by renaming `Screen` to `Display`, current `Window` to `Screen`,
and then adding `class [Main]Window < Widget`.

All mentioned classes `include` [EventHandler](https://github.com/crystallabs/event_handler) for events-based
behaviors.

### Positioning and Layouts

Widget positions and sizes work like in Blessed. They can be specified as numbers (e.g. 10), percentages (e.g. "10%"), both (e.g. "10%+2"), or specific keywords ("center", which has an effect of `50% - self.with_or_height//2`).

That model is simple and works quite OK, although it is not as developed as the model in Qt. For example, there is no way to shrink or grow widgets disproportionally when window is resized, and
there is no way to define maximum or minimum size. (Well, minimum size calculation does exist for resizable widgets, but only in the form which tries to find the real minimum size based on
contents rather than programmer's wish. (In Blessed, what we call "resizable" is called "shrinkable" even though it can also grow.))

Speaking of layouts, the one layout engine currently existing, `Widget::Layout`, is equivalent to Blessed's. It can arrange widgets in drid-like or masonry-like style.
There are no equivalents of Qt's `QBoxLayout`.

However, both the positioning and layout code is very manageable and adding Qt-like or other features is only a matter of very reasonable development time.
(Whether various layouts would then still inherit from `Widget` or not be based on it (like in Qt) is open for consideration.)

Finally, worth noting, there are currently some differences in the type of values supported for `top`, `left`, `width`, `height`, `align`, and `valign`. It would be
good if all these could be unified to accept the same flexible/unified specification, and if the list of supported specifications would even grow over time.
(One could pass a block or proc, in which case it'd be called to get the value.)

### Rendering and Drawing

Windows contain widgets. To make windows appear on screen with all the expected contents and current state,
one calls `Window#render`. This functions calls `Widget#render` on each of immediate children elements which
results in the final/rendered state reflected in internal memory.

At the end of rendering, `Window#draw` is called which makes any changes in internal state appear on the
screen. For efficiency, painter's algorithm is used, only changes ("damage") are rendered, and renderer
can optionally make use of CSR (change-scroll-region) and/or BCE (back-color-erase) optimizations
(see `OptimizationFlag`).

### Text Attributes

Generally speaking, to define foreground and background colors and attributes for strings, one can embed
appropriate escape sequences into the strings themselves or use Crystal's `Colorize` module.

Crysterm is interoperable with those two approaches, but also implements its own concept of "tags" in strings,
such as "{lightblue-fg} text in light blue {/lightblue-fg}". Tags can be embedded in strings directly, applied
from a Hash with `generate_tags`, and removed from a string with `strip_tags` or `clean_tags`.
Any existing strings where "{}" should not be interpreted can be protected with `escape_tags`.

### Testing

Run `crystal spec` as usual.

More specs need to be added.

One option for testing, currently not used, would be to support a way where all output (terminal
sequences etc.) is written to an IO which is a file. Then simply the contents of that file are
compared with a known-good snapshot.

This would allow testing complete programs and a bunch of functionality at once, efficiently.

### Documentation

Run `crystal docs` as usual.

## Thanks

* All the fine folks on Libera.Chat IRC channel #crystal-lang and on Crystal's Gitter channel https://gitter.im/crystal-lang/crystal

## Other projects

List of interesting or similar projects in no particular order:

Terminal-related:

- https://github.com/Papierkorb/fancyline - Readline-esque library with fancy features
- https://github.com/r00ster91/lime - Library for drawing graphics on the console screen
- https://github.com/andrewsuzuki/termbox-crystal - Bindings, wrapper, and utilities for termbox

Colors-related:

- https://crystal-lang.org/api/master/Colorize.html - Crystal's built-in module Colorize
- https://github.com/veelenga/rainbow-spec - Rainbow spec formatter for Crystal
- https://github.com/watzon/cor - Make working with colors in Crystal fun!
- https://github.com/icyleaf/terminal - Terminal output styling
- https://github.com/ndudnicz/selenite - Color representation conversion methods (rgb, hsv, hsl, ...) for Crystal
- https://github.com/jaydorsey/colorls-cr - Crystal toy app for colorizing LS output

Event-related:

- https://github.com/crystallabs/event_handler - Event model used by Crysterm
- https://github.com/Papierkorb/cute - Event-centric pub/sub model for objects inspired by the Qt framework
- https://github.com/hugoabonizio/event_emitter.cr - Idiomatic asynchronous event-driven architecture

Misc:

- https://github.com/Papierkorb/toka - Type-safe, object-oriented option parser
- https://github.com/eliobr/termspinner - Simple terminal spinner
- https://github.com/epoch/tallboy - Draw ascii character tables in the terminal
- https://github.com/teuffel/ascii_chart - Lightweight console ASCII line charts
