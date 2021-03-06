### Event loops

HTML and script in web pages use an event loop-driven model of computation. The
event loops for HTML are defined
[here](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop). Tasks
on an event loop live in one or more [task
queues](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue), and
each loop only has one task at a time. Event loops also have [microtask
queues](https://html.spec.whatwg.org/multipage/webappapis.html#microtask), which
are processed all-at-once at times when [microtask
checkpoints](https://html.spec.whatwg.org/#perform-a-microtask-checkpoint) are
specified to happen.

Microtask checkpoints happen immediately after script yields, and also at
various points where microtasks may have been enqueued for other reasons not
related to script. In such cases, there should always be a microtask checkpoint
before script has an opportunity to run.

While there can only be one task at a time on an event loop, tasks can execute
[in parallel](https://html.spec.whatwg.org/#parallelism) in some cases, and also
generate promises to mark their completion. These promises, which can be held
onto by script, run completion microtasks (see also [this spec
text](https://html.spec.whatwg.org/#integration-with-the-javascript-job-queue))
when the underlying (async) task is complete.

### The browser context (aka rendering) event loop

There are several event loops, but the most important is the [browsing
context](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context)
event loop. This event loop is special because it is where tasks related to the
following all occur:

* document-loaded scripts (as opposed to worker scripts), DOM, input event
* processing rendering updates

As it relates to rendering, the spec currently defines image decoding, video,
and HTMLCanvasElement.{toBlob,convertToBlob} to be in parallel.

However, any task which affects data observable by a script (i.e. specific to a
particular script realm, global or environment settings object) [must not run in
parallel](https://html.spec.whatwg.org/#event-loop-for-spec-authors) to other
tasks in the queue which may run the script. This means that:

* Script can not be run in parallel with script from the same realm
* DOM (which is script-observable) cannot be mutated in parallel with script

## Rendering: step 7 of the browsing context event loop

Step 7 of the [event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model), is the steps to get ready and "update the rendering". This is
spec'ed to potentially happen at the end of processing every task in a browsing
context event loop. However, it is also specified that these steps need not
happen after every task, and may be throttled to a specified framerate, or for
other reasons.

The update-the-rendering steps in the HTML specification are:

1. Resize
2. Scroll
3. Media queries
4. Update animations, fire animation events
5. Fire fullscreen events
6. Run animation frame callbacks
7. Update intersection observations
8. "Update the rendering" (draw things to the screen)

The spec has no concept of style or layout computation. It's assumed to be
synchronously implied by DOM changes, even before step 1 above happens.


# Rendering in parallel

The steps of rendering (i.e. steps 1-8 from the previous section) must happen in
series, and if rendering is updated they must all be run. In addition, the spec
does not say that these steps are in parallel with other tasks. However, as
discussed above, the only thing preventing tasks or steps from running in
parallel in principle is script observability, which is in turn due to the
inherently single-threaded nature of JavaScript.

Note also that *script observability* (a script program can see the difference)
is different than *user observability* (the person viewing the web site can see
something happening). Thus if we are willing to let the final rendering to the
screen be temporarily out of sync (i.e. what the user can see is different than
what the script last saw), there is an opportunity for parallelism. (Note that
the HTML spec makes no reference to this possibility, though it does not
preclude it either.)

# Chromium rendering behavior

Chrome implements step 7 of the browsing context event loop via what it calls a
"BeginMainFrame" task. BeginMainFrame tasks are scheduled at 60Hz, but only if
some rendering-related state has changed that necessitates the rendering update.
For Chromium, steps 1-6 from the [event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model) do not necessarily run any task in particular. However,
Chromium resolves promises of image decode tasks (step 1 below), and dispatches
input events (step 3) that have need to be rendering-time aligned. In addition,
some syncing happens from the compositor thread (step 2) The processing time of
these cases are not currently spec'd.

Chromium steps:

1. Add callbacks for completed image decodes to microtask queue
2. Synchronize compositor thread scroll and scale to main thread
3. Dispatch input events which are requestAnimationFrame aligned (therefore
calling into script)
4. Update autoscroll animations
5. Update other scroll animations
6. Update snap fling animations
7. Update SVG animations
8. Update media sources based on media query changes
9. Dispatchdeclarative animation events
10. Dispatch fullscreen events
11. Call requestAnimationFramecallbacks
12. Update "validation message" overlay
13. Update style [figure out ComputedStyles of DOM elements]
14. Update layout [size and place DOM on relative to a screen viewport]
15. Adjust for scroll anchoring
16. Notify resize observers
17. Notify sub-frames of change to their viewport size & location
18. Commit pending selection ([spec](https://w3c.github.io/selection-api))
19. Update Blink compositing [decide texture backing strategy for DOM]
20. Update Blink property trees and invalidate paint
21. Update intersectionobservations
22. Update paint
23. Update touch event regions and main-thread
scrolling reasons
24. Invalidate raster
25. Commit paint output to compositor
thread

# Parallel aspects to Chromium behavior

(Composited scroll, etc.) TODO



