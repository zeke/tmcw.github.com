---
layout: post
title: CSS For Maps
categories:
- blog
---

[The first commit](https://github.com/mapbox/carto/commit/dfb4ce97425b72d241c3da7c65f30bacd8bc1a33)
to what became [CartoCSS](http://mapbox.com/carto/api/2.1.0/) (but was temporarily named 'mess', 'carto.js', and then 'carto') was made on
December 1, 2010. Since then it's gotten a lot of developer love - most of
its speed is thanks to [Konstantin](https://kkaefer.com/)'s genius and recent
refinements are thanks to [Dane](http://dbsgeo.com/).

<img src='/graphics/map_css_gif.gif' />

It's been a while, and, thanks to its inclusion in [TileMill](http://mapbox.com/tilemill/),
a lot of people have used CartoCSS. It's about time for some reflection.

## Introduction

For the most part, web maps are images: when you go to [Google Maps](http://maps.google.com/)
or make a map with [TileMill](http://mapbox.com/tilemill/), you are seeing
[PNG](http://www.libpng.org/pub/png/) image files which have been rendered
by a renderer. In the case of TileMill, that renderer is [Mapnik](http://mapnik.org/),
a successful open-source project - Google Maps is proprietary technology.

These maps are styled; roads have a color, width, label style, halos, and
so on - very complex styles, that are made 'once' - by assigning a type to
a style and letting that dictate the world. And so this styling is rarely
done with a [WYSIWYG](http://en.wikipedia.org/wiki/WYSIWYG) interface like
Adobe Photoshop - it's more likely similar to coding.

Renderers thus support style formats. Mapnik natively supports XML stylesheets,
which are great for control, but can be difficult to learn and write by
hand, especially for people who are mainly designers and artists rather than
programmers.

This article is about a specific style format called CartoCSS which extends
the metaphor of CSS to maps. The kernel of the idea was devised by
[Mike Migurski in 2008](http://mike.teczno.com/notes/cascadenik.html), and the
most recent iteration was done by myself and the team at [MapBox](http://mapbox.com/)
starting in 2010. This post is about the evolution of CartoCSS, thoughts
on whether this metphor of CSS-to-maps is a [leaky abstraction](http://en.wikipedia.org/wiki/Leaky_abstraction)
or if its usefulness outweights its awkwardness.

## The Why of CartoCSS

TileMill was originally implemented in [Python](https://github.com/mapbox/tilemill/commit/5b870b04d8e2a75855f7523a1937f5bad0eb0ae8).
We were going to do it with [Tornado](http://www.tornadoweb.org/), because
[TileLive](https://github.com/mapbox/TileLive) used Tornado and [Mapnik](http://mapnik.org/)'s
Python bindings were strongest.

As time went on, we decided to make it multi-platform and saw that an
async-in-sync framework like Tornado would be unsustainable. And so we decided
on [node.js](http://nodejs.org/), still then a very early project, because
of its design & all-async approach.

But, as I said, originally TileMill was in [Python](http://www.python.org/),
and that made its use
of [Cascadenik](https://github.com/mapnik/Cascadenik) possible.
Cascadenik, by [Mike Migurski](http://mike.teczno.com/) and the [Stamen](http://stamen.com/)
team, is entirely the inspiration for Carto.

However, as time went on, we found a few things to be desired. I was chasing
after better error reporting - since TileMill would report errors in a
GUI and ideally give great feedback for getting novice users up to speed.
Cascadenik's performance on large stylesheets was also lacking at the time.
And, finally, of course, bridging the gap between a Python stylesheet processor
and a node.js application was awkward and problematic.

## More Or LESS CSS

The CartoCSS implementation is based off of [LESS CSS](http://lesscss.org/),
one of the strongest CSS pre-processors around. I have no regrets around this
decision - the LESS parser has been fast enough to never be a bottleneck,
and its general AST technique is fairly clean.

CartoCSS, like Cascadenik looks like CSS, which is a language friendly enough
to fit into with 'it isn't programming' mold of HTML: specifically, lots of
people use CSS.

However, it's important to note that CartoCSS **isn't CSS**. To be clear, I'm
happy when people come away with the impression that CartoCSS is CSS, and that
TileMill is using CSS to style things in-browser. If we can create that level
of magic, it's wonderful - and taking advantage of the CSS syntax familiarity
to reduce people's self-destructive "I can't do this" reaction is huge.

## Not Quite CSS

CartoCSS is not CSS in a few ways.

First, [Mapnik](http://mapnik.org/) does not implement cascading - Carto does.
Internally, CartoCSS implements [CSS specificity](http://www.w3.org/TR/CSS2/cascade.html),
collapses and coalesces rules, and essentially plays the role of a web browser.
This is where most of the performance gain is from Cascadenik (in doing this intelligently)
but also where most of the effort goes in maintaining it.

Second, Mapnik follows a painter's model, not a document model. What this means
is that in CSS, you can have code like:

{% highlight html %}
<style>
div { border: 1px solid #000; }
</style>
<div>foo</div>
{% endhighlight %}

You give the div _its border_: the div has one potential border, which can
be styled and turned on and off with `border: none`.

The painter's model is the reverse: there are potentially infinite borders,
fills, markers, and so on, and you assign them to elements. In mock-HTML,
it would be like:

{% highlight html %}
<style>
border[div] {
    width: 1px;
    style: solid;
    fill: #000;
}
</style>
<div>foo</div>
{% endhighlight %}

Also, the HTML 'order' is no longer. HTML stacks styles - borders are around
fills, paddings are within them. Maps can have borders behind or in front
of fills, and multiple borders in any order.

What does this mean for a CSS pre-processor for maps? You could simply
restrict this power from users of the pre-processor, but we decided not to -
after all, TileMill is used internally for the [MapBox Streets](http://mapbox.com/tour/design/)
product, so it needs to scale to extreme-pro use cases.

CartoCSS does this with attachments:

{% highlight css %}
#world {
    ::borderback { line-width: 5; }
    line-color: #f00;
    line-width: 2;
}
{% endhighlight %}

The attachment syntax (stolen from CSS pseudo-elements) lets you say that
'for this style I'm going to have multiples of a symbolizer'. Cacadenik
approached this problem differently - you had `line-outline` and `line-inline` -
like preset attachments that gave you a few bonus symbolizers for specific
purposes. Cascadenik now supports attachments, so there's no real difference
here.

The idea of the painter's model not having a guaranteed order also affects
how ordering in the language works:

{% highlight css %}
#world {
    polygon-fill: #0f0;
    line-width: 2;
}
/* this has a different order in the rendering */
#world {
    line-width: 2;
    polygon-fill: #0f0;
}
{% endhighlight %}

Another big difference is that with the data model of maps, **there is no nesting**.

Nesting is essential in HTML & CSS - HTML is a document _tree_ and we very
often uses classes like `p a` to style links within paragraphs differently
than links links in menus. Expressing this nesting is one of the major
advantages of using LESS on top of CSS - instead of writing

{% highlight css %}
#world {
    background: red;
}
#world .link {
    color: blue;
}
{% endhighlight %}

You can write

{% highlight css %}
#world {
    background: red;
    &.link {
        color: blue;
    }
}
{% endhighlight %}

This leads to rather better code structuring. Since every object in Mapnik
is an atom - it doesn't contain anything else - we use LESS's nesting exclusively
for combination and drop some of its syntax.

{% highlight css %}
#world {
    background-color: red;
    .link {
        line-color: blue;
    }
}
{% endhighlight %}

Turns into

{% highlight css %}
#world {
    background-color: red;
}
#world.link {
    line-color: blue;
}
{% endhighlight %}

Note the lack of the space between `#world` and `.link` - it's combination,
not nesting.

## Evolution

A lot has changed with CartoCSS over its life. One of the biggest changes
was improving versioning with Mapnik: both CartoCSS and Mapnik have versions,
and the CartoCSS compiler can take a Mapnik version and compile (and verify)
to it.

A few scattered changes:

Carto acquired nicer ways of handling fields in datasources from Mapnik: what
used to be `text-name: "[FOO]"` can now be `text-name: [FOO]`, and using
expressions like `marker-width: [FOO]` can easily also support
`marker-width: [FOO] * 2`. This is actually kind of an insane change, because
it needs to push expressions back into Mapnik, and can't do this purely. You
can use variables, like `marker-width: [FOO] * @scale`, and CartoCSS does
the right thing.

'Instances' were added by Konstantin and allow you to do:

{% highlight css %}
#roads {
  casing/line-width: 6;
  casing/line-color: #333;
  line-width: 4;
  line-color: #666;
  }
{% endhighlight %}

Instead of classic casings, in which one layer of casings displays under all
centerlines, this will show a casing under each line, possibly overlapping
with another line underneath.

FontSets are a cool Mapnik features that we support like

{% highlight css %}
text-face-name: "Georgia Regular", "Arial Italic";
{% endhighlight %}

Behind the scenes this is intelligent; if a lot of rules use the same list
of fonts, CartoCSS does its best to condense all of the generated XML
fontsets into one reference.

## MapCSS

Cascadenik is, as far as I know, the first CSS-like map styling language -
aka 'the grandaddy'.

**[MapCSS](http://wiki.openstreetmap.org/wiki/MapCSS)** is used by the
[OpenStreetMap](http://www.openstreetmap.org/) project in quite a few places -
[Potlatch 2](http://wiki.openstreetmap.org/wiki/Potlatch_2), [JOSM](http://josm.openstreetmap.de/),
and an in-browser renderer called [Kothic JS](https://github.com/kothic/kothic-js).
The main difference of MapCSS is that it is relatively data-specific -
while CartoCSS targets any relational datasource with `key=value` pairs
that can be filtered upon, MapCSS is very specific to the OSM data model:

* You refer to OSM data types like `way`, `node`, and `relation` rather than
  tables or datasources.
* Filter syntax supports tagging like `way[highway=primary]`
* There are a few features like `eval()` and `z-index` that are implementation-specific.

## The Future in Browsers

There are scattered attempts at supporting CartoCSS in-browser - the furthest
advanced being [Vizzuality's Vecnik project](https://github.com/Vizzuality/VECNIK).

While my heart immediately wanted to just go straight CSS and used [SVG](http://en.wikipedia.org/wiki/Scalable_Vector_Graphics)
for maps, this doesn't seem to be a viable option - given that SVG mainly
uses the same rendering model as HTML (with odd exclusions like no z-indexes),
it's very difficult to render complex scenes without lots of elements and hacks.
It's no wonder that the only viable vector street maps in browsers -
Google's MapsGL - use a 'proprietary' renderer based on [WebGL](http://en.wikipedia.org/wiki/WebGL)
instead of 2D Canvas or SVG.

But, for simple maps SVG and CSS can cut it - as they do in
[the iD editor project I've been recently working on](https://github.com/systemed/iD).

## The Future Elsewhere

Together with [Stamen](http://stamen.com/), [Google](http://www.google.com/),
[OpenGeo](http://opengeo.org/), and some community members I've met to talk about
what to do about a cascading style sheet language for more common maps.
Besides CartoCSS, there are scattered smaller languages and then a few common
but unliked languages like [SLD](http://blog.geoserver.org/2010/04/09/sld-cookbook/).

It's going to be difficult - CartoCSS is made to support everything that Mapnik
supports - to expose its full power, and to make a 'subset' that works
with everything will be difficult to communicate, just as CSS has had
a terrible struggle with [vendor prefixes](http://www.alistapart.com/articles/the-vendor-prefix-predicament-alas-eric-meyer-interviews-tantek-celik/).

But, it's a worthwhile goal and will happen. It will be especially interesting
to think of CartoCSS as a simple language that can be generated, or as
the source of a much simpler subset. As far as its future with Mapnik,
[carto-parser](https://github.com/rundel/carto-parser) is an incredible
effort to do native parsing - so we can drop the [node.js](http://nodejs.org/)
requirement for people who want stylesheet-ed maps in Mapnik.
