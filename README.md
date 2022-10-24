# Cross-Origin isolation with popups

This repository is dedicated to finding solutions to the cross-origin isolation with popups problem.

## Authors:
[Arthur Hemery](https://github.com/hemeryar)

## To Participate
https://github.com/whatwg/html/issues/6364

## Table of Contents

[You can generate a Table of Contents for markdown documents using a tool like [doctoc](https://github.com/thlorenz/doctoc).]

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- END doctoc generated TOC please keep comment here to allow auto update -->


## Terminology
In this explainer, we call _document_ what is rendered by any individual frame. A _page_ contains the top level document, as well as all of its iframe documents, if any. When you open a popup, you get a second page.

## Motivations
Currently for a document to be `crossOriginIsolated`, we need a `Cross-Origin-Opener-Policy: same-origin` header, and `Cross-Origin-Embedder-Policy: require-corp` or `credentialless`. The `COOP` value prevents any interaction with cross-origin popups.

This proposal main goal to be able to enable `window.crossOriginIsolated` for pages that interact with cross-origin popups, both the opener and the openee. This would allow the use of APIs gated behind `crossOriginIsolated` such as `SharedArrayBuffer` or the Memory Measurement API to be used more widely. These flows are especially important because they are common. Oauth and payments are typical examples.

A secondary objective is to build on the solution to improve security against XS-Leaks in general, by making a solution that could be suitable to a wide.

## Why does crossOriginIsolated require COOP: same-origin
For a page to be `crossOriginIsolated`, we need to be able to put it in its own process. This is the only thing that guarantees that the browser can actually make a page safe following Spectre. For a browser being "able to" means not breaking specifications, and in particular not restricting DOM scripting, which is impossible across processes. `COOP: same-origin` was designed to disable that capability for popups.

</br>

![image](resources/coop_basic_issue.png)  
_The basic case COOP solves. Without COOP, we have to put all the documents in the same process, because the popup and the iframe are of origin b.com and synchronously script their DOM._

</br>

## The COOP: restrict-properties proposal
Instead of completely removing the opener, we propose having a new `COOP` value, `restrict-properties` that only restricts the access of same-origin documents that do not share this `COOP` header value as well as top-level origin. In the above case, if the opener has `COOP: restrict-properties`, we do not allow access of the iframe to the newly opened popup. This makes it possible to put the two pages in different processes, and to enable `crossOriginIsolated` on the first page, as long as it also sets `COEP`.

We widen the usefulness of `COOP: restrict-properties` by also limiting cross-origin accesses to a very limited set of properties: {`window.closed` and `window.postMessage()`}. This is base on metrics research that shows that the overwhelming majority of sites only uses these two properties when interacting with cross-origin popups. This prevents almost all `WindowProxy` XS-Leaks.

## COOP: restrict-properties and other values
`COOP` works by comparing values and producing a decision about what to do with the opener. We propose that:

A page with `COOP: restrict-properties` can navigate to or open:
* same-origin `COOP: restrict-properties` with a full opener.
* `COOP: unsafe-none` with a restricted opener.
* cross-origin `COOP: restrict-properties` with a restricted opener.
* Any other `COOP` value with a severed opener.

A page with `COOP: restrict-properties` can be opened or navigated to by:
* same-origin `COOP: restrict-properties` with a full opener.
* `COOP: unsafe-none` with a restricted opener.
* cross-origin `COOP: restrict-properties` with a restricted opener.
* `COOP: same-origin-allow-popups` with a restricted opener, iff its in a newly created popup.

## Typical use case & the reversability characteristic
Because the ecosystem of popup flows is complex, we want `COOP: restrict-properties` to be as little intrusive as it can be.

</br>

![image](resources/auth_provider_flow.png)  
_An authentication provider uses a navigation flow to provide login with many different providers. We do not want one of them setting `COOP: restrict-properties` limiting the interactions between my-website.com and the provider._

</br>

Since we do not sever the opener, there is no need to make a `COOP: restrict-properties`'s action irreversible, contrary to `COOP: same-origin` which is enforced definitively, even on redirects.

This make `COOP: restrict-properties` completely transparent, unless it is used for one two pages directly interacting with each other. we call that the _reversibility_ of `COOP: restrict-properties`.

## Security interlude on the same-origin policy
Our proposal creates an unprecedented possibility: that two same-origin documents can reach one another but not have DOM access. However DOM access is not the only thing that is gated behind same-origin restrictions. We audited the spec to produce a [list](https://docs.google.com/spreadsheets/d/1e6LakHSKTD22XEYfULUJqUZEdLnzynMaZCefUe1zlRc/) of all places with such checks. Some points worthy of attention:

* The location object is quite sensitive and many of its methods/members are same-origin only. It is purposefully excluded from the list of allowed attributes by `restrict-properties`. We do not think we should allow a normal page to navigate a `crossOriginIsolated` page.
* For similar reasons name targeting does not work across pages with `COOP: restrict-properties`, unless they also share their top-level origin.
* Javascript navigations are a big NO. They mean executing arbitrary javascript within the target frame. There should be no way to navigate a frame across the `COOP: restrict-properties` boundary given the restrictions above are put in place.

## COOP: restrict-properties and subframes opening popup
What happens when an iframe in a `COOP` page opens a popup? The initial empty document created always inherits the origin of the iframe, while we would like `COOP` to be inherited from the top-level document. This can create dangerous discrepencies where we end up with a `crossOriginIsolated` initial empty document of an arbitrary origin.

For `COOP: same-origin` we solved this problem by setting no-opener on any popup opened from an iframe that is cross-origin to its top-level document. We do the same for `COOP: restrict-properties`.

## The window.name problem
When we navigate to a `COOP: restrict-properties` page and then to a `COOP: unsafe-none` page, we need to make sure no state remains from the previous context, to limit XS-Leaks. `Window.name` can be set by a `crossOriginIsolated` page and it could expose information to the next site.

Instead, when doing a restricted swap, we get a fresh name if that's for a context we haven't seen yet, or reuse the `window.name` property that was set for that frame in that context. Each window has a separate name in all the different contexts.

</br>

![image](resources/name.png)  
_In that example all the documents with origin A.com can set and target the window.name property. It is in a different context from the B.com's page, so we stash the name when navigating. B.com free to set its own name and use it in its context. When we navigate back to A.com we reuse the stashed name._

</br>


## Notes on COOP: restrict-properties reporting
The COOP infrastructure can be used to report access to cross-origin properties that would not be postMessage nor closed. As for usual COOP reporting, DOM access that become restricted will not be able to be reported, and only cross-origin available properties other than postMessage or closed will be reported.

This is a fundamental limitation, because reporting synchronous DOM access would require a check on every Javascript access that would have unacceptable performance impact.

## Stretch - COOP: restrict-properties as a default candidate
Given the properties that we have developed in this explainer, we believe `COOP: restrict-properties` could, in the future, make a candidate to replace `COOP: unsafe-none` as a default. Some design decisions were also made to make it as plausible as possible. Some consequences of making it default would be:
* Most cross-origin properties are inaccessible to and from cross-origin popups, unless you manually set `COOP: unsafe-none`.
* Pages would only have to manually set `COEP` to be `crossOriginIsolated`.
* Popups opened from cross-origin iframes would always be no-opener, unless `COOP: unsafe-none` or `COOP: same-origin-allow-popups` is specified.


## Stakeholder Feedback / Opposition

- Firefox: No Signals
- Safari: No Signals
- TAG: Ongoing review in https://github.com/w3ctag/design-reviews/issues/760

## References & acknowledgements
Many thanks for valuable feedback and advice from: Alex Moshchuk, [Anne VK](https://github.com/annevk), [Artur Janc](https://github.com/arturjanc), [Camille Lamy](https://github.com/camillelamy), [Charlie Reis](https://github.com/csreis), [David Dworken](https://github.com/ddworken).
