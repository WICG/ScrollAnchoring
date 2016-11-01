# Scroll Anchoring

Scroll anchoring is an
[intervention](https://github.com/WICG/interventions)
that adjusts the scroll position to reduce visible content "jumps".

## Overview

Today, users of the web are often distracted by content moving around due to
changes that occur outside the viewport.  Examples include script inserting an
iframe containing an ad, or non-sized images loading on a slow network.

Historically the browser's default behavior has been to preserve the absolute
scroll position when such changes occur.  This meant that to avoid shifting
content, the webpage had to take pains to reserve the correct amount of space on
the page for anything that will load after the initial render (for example, by
declaring sizes on `img` tags).  This is not always practical, especially with
third-party content, and few websites do it consistently.

Scroll anchoring aims to minimize surprising content shifts by scrolling to
follow the movement of visible page elements.

## Draft Spec

[Draft spec](https://skobes.github.io/ScrollAnchoring)

## Design

Scroll anchoring works by selecting a DOM node (the *anchor node*) whose
movement is used to determine the adjustment to the scroll position.

When the anchor node moves, the browser computes its previous location L0 and
its new location L1 in the coordinate space of the scrolling content.  It then
reads the current scroll position S0 and computes a new scroll position S1:

  S1 = S0 + (L1 - L0)

By setting the scroll position to S1 at the same time that the anchor node moves
to L1, we preserve the location of the anchor node relative to the viewport, and
no "jump" is visible.

The anchor node may be either a text node or an element.

Conceptually, a new anchor node is computed whenever the scroll position
changes.  (As a performance optimization, the implementation may wait until the
anchor node is needed before computing it.)

## Anchor Node Selection

We aim to select an anchor node that is deep in the DOM and close to the top
edge of the viewport.  This maximizes the amount of offscreen content whose
changes we will successfully compensate for, while avoiding undesired scrolling
when important content loads within the viewport (for example, upon clicking a
"Read More" link).

The current algorithm to select an anchor node for a scrollable area (either a
document or an element with scrollable overflow) is as follows:

* Walk the DOM within the scrollable area.

* For each element E encountered by the walk,

    - If E is `position: fixed`, or if E is `position: absolute` and E's
      containing block is an ancestor of the scrollable area, or if E has
      `overflow-anchor: none` (see Exclusion API), skip over E and its
      descendents.

    - Otherwise, compare its bounds to the scrollable area's visible region.

        - If E is fully visible, terminate the walk and use E as the anchor
          node.

        - If E is fully clipped (not in the viewport), skip over E and its
          descendents.

        - If E is partially visible, mark it as a "candidate" and descend into
          its children.  If the walk reaches the end of E without finding
          another anchor node, use E as the anchor node.  (We prefer deeper
          nodes in this case to avoid the failure mode of content being inserted
          inside the anchor node but outside the viewport.)

## SANACLAP

Initial testing revealed that scroll anchoring often performed undesired scroll
adjustments in response to changes in the computed style of the anchor node or
one of its ancestors.  For example, a map panning interface might update a CSS
transform in response to mouse movements, or a sticky-header implementation
might adjust padding on the body element.

The SANACLAP principle ("**s**uppress if **a**nchor **n**ode **a**ncestor
**c**hanged a **l**ayout-**a**ffecting **p**roperty") aims to suppress these
undesired scroll anchoring adjustments.

Under the SANACLAP principle, if the anchor node, or any ancestor of the anchor
node up to and including the scrollable element (or document), has any of a
certain set of CSS properties modified, any scroll anchoring adjustment that
would have resulted from that modification is suppressed.

Currently the set of properties that trigger a SANACLAP suppression are:

* `left`, `top`, `right`, `bottom`
* margin and padding properties
* `width`, `min-width`, `max-width`, `height`, `min-height`, `max-height`
* `position`
* `transform`

## Flow Change Suppression

If the DOM changes made by a scroll event handler alter the position of the
anchor node, we may encounter feedback loops in which scroll anchoring and
the scroll event handler trigger each other ad infinitum.

A common way in which this occurs is when a header is fixed-position below a
scroll offset, and the anchor node (in subsequent content) moves due to the
header leaving the [normal flow](https://drafts.csswg.org/css-position-3/#nf).

(The SANACLAP principle doesn't help here, because the header is not an ancestor
of the anchor node.)

To address this case, we suppress any scroll anchoring adjustment arising from a
change in the computed `position` style of any element in the scroller (not just
those in the ancestor path) that pulls the element in or out of normal flow.

The values `absolute` and `fixed` are considered out-of-flow for this purpose;
all others are considered in-flow.

## Exclusion / Opt Out API

Scroll anchoring aims to be the default mode of behavior when launched, so that
users benefit from it even on legacy content, but we also want to give
developers control over the feature if and when they need it.

In particular, developers should be able to use CSS

* to disable scroll anchoring in part or all of a webpage (opt out), or
* to exclude portions of the DOM from the anchor node selection algorithm
  (exclusion).

We propose a CSS property `overflow-anchor` whose values have the following
meaning for some element E:

* `overflow-anchor: auto` declares that the DOM subtree rooted at E is
  eligible to participate in the anchor node selection algorithm for any
  scrollable area created by E or an ancestor of E.  This is the default.

* `overflow-anchor: none` declares that the DOM subtree rooted at E should be
  skipped by the anchor node selection algorithm for any scrollable area created
  by E or an ancestor of E.

Note the following implications of the above semantics:

* If `overflow-anchor: none` is applied to a descendant D of a scroller S, then
  S will skip over D when selecting an anchor node, but can still perform scroll
  anchoring adjustments based on anchor nodes outside of D.

* If `overflow-anchor: none` is applied to a scroller S, then S will not perform
  any scroll anchoring adjustments.

* If `overflow-anchor: none` is applied to the root element (`html`), the
  document-level scrollable area will not perform any scroll anchoring
  adjustments.

The `overflow-anchor` property is also proposed (with different values) for
[CSS Sticky Scrollbars](http://tabatkins.github.io/specs/css-sticky-scrollbars/).
This feature is distinct from scroll anchoring, but shares the notion of
adjusting scroll position in response to content changes.

The `overflow-anchor` property is not inherited (but the effect of skipping a
subtree in the anchor node selection algorithm bears a resemblance to property
inheritance).

## History Scroll Restoration

The current implementation of scroll anchoring has a known limitation in its
interaction with history scroll restoration.

Browsing history entries are stored with (absolute) scroll offsets, which are
restored during back/forward navigation.  Typically the offset is recorded after
everything on the page has finished loading, but the restoration is done before
the load completes (since a jump when the load completes would be jarring).

This creates a problem when scroll anchoring makes adjustments during page load,
since we may end up at a different offset by the time the load completes.

In the current architecture we have no way to perform correct scroll restoration
and also avoid visible jumps during page load.

One way to solve this is for the browsing history entry to store something
analogous to a CSS selector for the scroll anchor node, instead of storing an
absolute scroll offset.

## Implementation Status

A prototype of scroll anchoring is behind a flag in Google Chrome and can be
enabled from the "about:flags" page.  The implementation is tracked in
[Issue 558575](http://crbug.com/558575) and its dependencies.

Mozilla has discussed a similar feature in
[Bug 43114](https://bugzilla.mozilla.org/show_bug.cgi?id=43114).
