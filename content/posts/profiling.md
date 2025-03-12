+++
title = 'Profiling Clojure and Rust'
date = 2025-03-11
draft = false
math = false
type = "post"
+++


# Profiling Clojure and Rust Quasicrystals

I rewrote my [Clojure implementation](/posts/clojurequasi/) of my quasicrystal image generator to [Rust](/posts/rustquasi/) for performance reasons. Before doing this I did try a couple of things to improve the Clojure performance, but after looking at a profile of the Clojure code, it became obvious that my fixes weren't addressing the core issue. I then decided to move to Rust because Rust seemed like a better tool for the job, and I wanted to learn more about it.

In this article we'll go over some of the things I tried to speed up my clojure code, and take a look at a flamegraph of the Clojure and Rust code to see where the performance (or lack thereof) comes from.

## Clojure performance fixes

As a baseline, running the sample code for the Clojure version takes about 45 seconds on my machine.

Before looking at a profile of the performance, I tried figuring out what was taking the most time. I figured since each pixel of each frame required several math function calls, that'd be a good place to start.

The first thing I did was memoize everything I could. Memoization just adding a lookup table to a function. The function is called, and the inputs/outputs are stored in a table. Then when it's called with those inputs again, the table is used instead of redoing the function. Generating these images involves a lot of math, specifically `sin`, `cos`, and `asin` function calls. A lot of the inputs are repeated, so I thought I could save some time memozing them. This did seem to be a little faster, but maybe not - the difference wasn't statistically significant.


Next I tried using different JVM math functions like `fastmath` and the code shared in this JVM game dev forum [thread](https://jvm-gaming.org/t/extremely-fast-sine-cosine/55153). Neither of these had an impact.

After some more investigation, I realized the JVM was probably already using the built in trig functions of my processor - so optimizing these was probably never going to see much if any speed increase.

## Clojure Profiling

After realizing I couldn't really optimize the math part, I decided to look at a profile of the performance. I used the instructions on [clojure-goes-fast](https://clojure-goes-fast.com/kb/profiling/clj-async-profiler/basic-usage/) to generate a flamegraph of my code:

![5 Clojure code flamegraph](/profiling/clojure_flamegraph.jpg)

From looking at this I realized the bulk of the time wasn't spent on math but actually storing the pixels or possibly introp with the JVM. To draw the images I'm using the `Graphics` JWT library. For each pixel I set the color and use `fillRect` to draw it. This makes it easy to set zoom levels, but is probably very inefficient - the graphics library wasn't really meant to be used this way, it was just easy for me to implement.

You can see in the graph that the largest bars are `java.lang.reflect.Method.copy`. This is a method that copies the methods of one class to another. I'm not sure exactly what's calling this method so much, but I suspect it has something to do with my approach to drawing graphics.

This is where I decided I either needed to completely rework the drawing part of the code, or switch to a faster language. The graphics options for Clojure didn't seem great at the time, so I switched to Rust.

## Rust Profiling

The Rust code was immediately much, much faster, so I never really needed to do performance fixes. But just to compare, I profiled a run of the Rust version using [samply](https://github.com/mstange/samply). I also compiled Rust with `debug-infolevel = 1` as described [here](https://nnethercote.github.io/perf-book/profiling.html):

![5 Rust code flamegraph](/profiling/rust_flamegraph.jpg)

This run was much faster, about 2 seconds vs 45 for Clojure. There's not much to talk about with the graph - it pretty much just runs the main functions for the program and that's what takes the time. The pixels are written directly as bytes to memory then to my SSD, and there's no JVM overhead to slow things down.

## Conclusions

It's easier to write fast graphics code in Rust than Java/Clojure. However the implementation of the Clojure version leaves a lot to be desired, I wouldn't be surprised if it's possible to get them pretty close to each other with more work. The Rust version was a lot harder to implement, but I suspect doing a fast Clojure version would be similar difficulty. That would probably involve writing the pixels to a buffer as bytes, then writing that to disk as an image - rather than using the Java graphics library to write a bunch of rects to a virtual graphics buffer.

Basically Rust forces you to do things the hard but performant way, whereas Clojure lets you do things easy but non-performant. Overall I like both languages, but they each have their niche.

