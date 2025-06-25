# HTTP response bundles

The intuition is this: Embedding data is faster than not embedding it on first load, and reducing round trips frequently boosts performance, especially on connections with high bandwidth-delay products with lots of smaller resources.

## Scenario

Consider this: a 100KB page, a 50KB script that loads a 100KB partial translation file, a 50KB stylesheet that proceeds to load 20 10KB images, and 5 1MB images in the document itself, for a total of 5.52MB raw.

- The proposed embedding's overhead is negligible at only 232 bytes, so the 5.52MB overhead remains accurate.
- There's two bounds to latency in the below calculations: 100KB page -> 5 1M images and 100KB page -> 100KB script/CSS -> 300KB dependencies.

If round-trip ping is 100ms and a connection is 100 Mbps (think: same-region low-end broadband):

- Opening a TCP connection requires a minimum of 600ms, and opening a QUIC connection 200ms.
- A response bundle with everything embedded would load in about 640ms. The page could be fully interactive without images after about 240ms, and the first image could come in about 80ms after that.
- Splitting the downloads would require about 840ms. The page could be fully live and rendered without images after about 640ms, and first couple images would have already come in.

If round-trip ping is 100ms and a connection is 1 Gbps (think: same-region fiber):

- Opening a TCP connection requires a minimum of 600ms, and opening a QUIC connection 200ms.
- A response bundle with everything embedded would load in about 250ms. The page could be fully interactive without images after about 210ms, and the first image or two could come in the next frame.
- Splitting the downloads would require about 604ms, being purely bound by network delay. All images would arrive while the script/CSS dependency responses are still pending.

If round-trip ping is 500ms and a connection is 5 Mbps (think: 4G):

- Opening a TCP connection requires a minimum of 3s, and opening a QUIC connection 1s.
- A response bundle with everything embedded would load in about 9.8s. The page could be made interactive without images after about 1.8s, and the first image could come in about 1.6s after interactive.
- Splitting the downloads would require about 10.8s. The page could be interactive after about 3.8s, and the first couple images would have already come in.

If round-trip ping is 600ms and a connection is 8 Mbps (think: geosync satellite):

- Opening a TCP connection requires a minimum of 3.6s, and opening a QUIC connection 1.2s.
- A response bundle with everything embedded would download in about 6.7s. The page could be made interactive without images after about 1.7s, and the first image could come in about 1s after interactive.
- Splitting the downloads would require about 7.6s. The page could be interactive without images after about 4.1s, and the first few images would have already come in.

Now, let's suppose those 1MB images were 100KB instead, for a total of 1.02MB raw.

- The proposed embedding's overhead is still negligible, so the 1.02MB overhead remains accurate.
- There's two bounds to latency in the below calculations: 100KB page -> 5 1M images and 100KB page -> 100KB script/CSS -> 300KB dependencies.

If round-trip ping is 100ms and a connection is 100 Mbps (think: same-region low-end broadband):

- Opening a TCP connection requires a minimum of 600ms, and opening a QUIC connection 200ms.
- A response bundle with everything embedded would load in about 280ms. The page could be fully interactive without images after about 240ms, and the first image or two could come in the next frame.
- Splitting the downloads would require about 840ms. The page could be fully live and rendered after about 640ms, and all the images would've already arrived.

If round-trip ping is 100ms and a connection is 1 Gbps (think: same-region fiber):

- Opening a TCP connection requires a minimum of 600ms, and opening a QUIC connection 200ms.
- A response bundle with everything embedded would load in about 210ms. The page could be fully interactive within 1-2 frames of the data being received.
- Splitting the downloads would require about 604ms, being purely bound by network delay. All images would arrive while the script/CSS dependency responses are still pending.

If round-trip ping is 500ms and a connection is 5 Mbps (think: 4G):

- Opening a TCP connection requires a minimum of 3s, and opening a QUIC connection 1s.
- A response bundle with everything embedded would load in about 2.6s. The page could be made interactive without images after about 1.8s, and the first image could come in about 160ms after load.
- Splitting the downloads would require about 3.8s, being purely bound by network delay. All images would arrive while the script/CSS dependency responses are still pending.

If round-trip ping is 600ms and a connection is 8 Mbps (think: geosync satellite):

- Opening a TCP connection requires a minimum of 3.6s, and opening a QUIC connection 1.2s.
- A response bundle with everything embedded would download in about 2.2s. The page could be made interactive without images after about 1.7s, and the first image could come in about 100ms after load.
- Splitting the downloads would require about 4.1s, being purely bound by network delay. All images would arrive while the script/CSS dependency responses are still pending.

## Other motivators

For offline self-sufficient pages, ordinary data URLs are the typical go-to, but they're absolutely terrible at duplication, and compression does not fully compress it away.

Many libraries rely on URLs. Pages commonly rely on URLs. This would give browsers a way to save them to single pages without either hacking `fetch`/etc or mucking with URLs, by simply saving the resources requested.

And finally, service workers are great, but shouldn't be necessary for an offline SPA. And it should be possible to integrate a service worker (and just workers in general) into a fully offline page without anything like `eval` or an impossible-to-update `data:`.

## Proposal

So, instead, I want to see about declaring assets in the initial download, independent of version and without needing complicated HTTP push rules that only half work and have very little backend-to-CDN support.

> Note that ease of adoption is among my top priorities here.

These serve three big purposes:

1. Web apps could be bundled and distributed very easily as single files. This simplifies CDN serving and updates, it speeds up install and load times for stuff like Electron apps, and it provides a mechanism for distributing web app bundles for mobile app stores and other situations where visiting servers isn't practical.
2. Servers could preemptively send database responses it knows will be immediately needed, without having to hope that HTTP/2 Push will be supported down the line.
3. Bundlers and CDNs can easily select which assets to serve up first and optimize delivery to ensure the page becomes visible as fast as possible. For instance, opening a CSS stream at the beginning and sending the content after its matching `<link>` or `@import` (so the CSS parser streams without buffering), sending fonts similarly around a `@font-face`, and sending critical image metadata right away while deferring the rest of the image to after the document.

### Format

Servers can return a file of type `multipart/http-response-bundle` (`.htrb`). This is strictly more powerful, as it also lets CSS fonts and JS libraries supply most things right out of the gate when loaded as scripts and stylesheets, and it also massively cuts down on the wire overhead compared to `data:` URLs.

Structure:

- 32-bit magic `\0htb`
- 32-bit version ID, initially 0
- LEB128 template count
- Sequence of request/response template size infos:
    - LEB128 request header pattern length
    - LEB128 response header length
    - LEB128 body length
- Sequence of request header pattern HPACKs
- Sequence of response header HPACKs
    - These should *not* include any `Content-Length`. That is implicit in the template size infos.
- Sequence of response bodies
 
The request header patterns would all share an HPACK decoder context, and the response headers would share another HPACK decoder context. (The parser could just reuse the context across both and just reset after the first.) The decoder follows a static maximum dictionary size of 4096 bytes.

### Semantics

If the response for a URL navigation fetch starts with the magic number and it has no content type, it is to be parsed as an HTTP response bundle.

The main document's response headers are mixed with the bundle's response headers. CSP and other security-related headers in the high-level response constrain all resources in the bundle, including the main document and iframes contained within. Beyond that, response headers for the bundle are not combined with those of any given file.

Individual files may be compressed using `Content-Encoding` in the response header frame as applicable.

The first file whose request matches the browser's request is used as the main document. It's highly recommended to make this first, but this is not required. If the main document doesn't match any header, the browser proceeds as if it 404'd.

Path patterns can be specified using the pseudo-header `:pattern` inside request headers. Multiple templates can be matched by simply repeating this pseudo-header.

> SPAs can use this to perform dynamic matching for the main document.

This can be used for not just HTML, but also CSS and JS. Often, a JS file wants to also bundle JSON, WebAssembly, or binary data, and CSS files often wrap images, fonts, and the like. It's also possible that some future system finds value in it for other things that aren't even browser-related. So there's value in making this generic.

The first asset registration always wins. If two pending requests both concurrently fetch different response bundles, whichever one downloads last wins.

### Partial updates

Partial updates to a bundle are possible. In this case, a 204 response may be returned with only the changed files sent in the bundle. This is best managed with etags, `If-Modified-Since`, or similar.

## Possible questions

I know some of you might have some questions about this.

### Why binary?

1. It needs to be quickly parsed. Images have to be consumed fast in gigabit networks, and this ensures they can be loaded and processed quickly.
2. It's simple. Simple solutions tend to work out the best.
3. It doesn't look like HTML, so existing intermediaries that don't know about it won't be trying to muck with it.

### Why not HTTP Push?

1. Bundlers and bundle consumers can't target it.
2. You can't order stuff across streams over QUIC, and this is important for triggering certain optimizations.
3. CDNs don't generally expose hooks for it. It's treated as an implementation detail for the most part.
4. It requires network access.

Ultimately, in order to solve the problems of efficient delivery and offline access, you'd have to invent half of this format anyways. So might as well skip the middleman.

### Why make this just a sequence of request-response pairs? Why not do some form of stream-based method?

While using something like HTTP/2-style streams could allow for stuff like pre-sizing image elements, this simply doesn't require it.
