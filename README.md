# react-dom-stream

This is a React renderer for generating markup on a NodeJS server, but unlike the built-in `ReactDOM.renderToString`, this module renders to a stream. Streams make this library as much faster in sending down the first byte of a page than `ReactDOM.renderToString`, and user perceived performance gains can be significant.

## Why?

One difficulty with `ReactDOM.renderToString` is that it is synchronous, and it can become a performance bottleneck in server-side rendering of React sites. This is especially true of pages with larger HTML payloads, because `ReactDOM.renderToString`'s runtime tends to scale more or less linearly with the number of virtual DOM nodes. This leads to three problems:

1. The server cannot send out any part of the response until the entire HTML is created, which means that browsers can't start working on painting the page until the renderToString call is finished. With larger pages, this can introduce a latency of hundreds of milliseconds.
2. The server has to allocate memory for the entire HTML string.
3. One call to `ReactDOM.renderToString` can dominate the CPU and starve out other requests. This is particularly troublesome on servers that serve a mix of small and large pages.


This project attempts to fix the first these three problems by rendering asynchronously to a stream.

When web servers stream out their content, browsers can render pages for users before the entire response is finished. To learn more about streaming HTML to browsers, see [HTTP Archive: adding flush](http://www.stevesouders.com/blog/2013/01/31/http-archive-adding-flush/) and [Flushing the Document Early](http://www.stevesouders.com/blog/2009/05/18/flushing-the-document-early/).

My preliminary tests have found that this renderer keeps the TTFB nearly constant as the size of a page gets larger. TTLB can be  longer than React's methods (by about 15-20%) when testing with zero network latency, but TTLB can be lower than React when real network speeds are used. To see more on performance, check out the [react-dom-stream-example](https://github.com/aickin/react-dom-stream-example) repo.

## How?

First, install `react-dom-stream` into your project:

```
npm install --save react-dom-stream react-dom react
```

There are two public methods in this project: `renderToString`, `renderToStaticMarkup`, and they are intended as nearly drop-in replacements for the corresponding methods in `react-dom/server`.

### Rendering on the server

To use either of the server-side methods, you need to require `react-dom-stream/server`.

#### `Readable renderToString(ReactElement element)`

This method renders `element` to a readable stream that is returned from the method. In an Express app, it is used like this (all examples are in ES2015):

```javascript
import ReactDOMStream from "react-dom-stream/server";

app.get('/', (req, res) => {
	// TODO: write out the html, head, and body tags
	var stream = ReactDOMStream.renderToString(<Foo prop={value}/>);
	stream.pipe(res, {end: false});
	stream.on("end", function() {
		// TODO: write out the rest of the page, including the closing body and html tags.
		res.end();
	});
});
```

Or, if you'd like a more terse (but IMO slightly harder to understand) version:

```javascript
import ReactDOMStream from "react-dom-stream/server";

app.get('/', (req, res) => {
	// TODO: write out the html, head, and body tags
	ReactDOMStream.renderToString(<Foo prop={value}/>).on("end", () =>{
		// TODO: write out the rest of the page, including the closing body and html tags.
		res.end();
	}).pipe(res, {end:false});
});
```

#### `Readable renderToStaticMarkup(ReactElement element)`

This method renders `element` to a readable stream that is returned from the method. Like `ReactDOM.renderToStaticMarkup`, it is only good for static pages where you don't intend to use React to render on the client side, and in exchange it generates smaller sized markup than `renderToString`.

In an Express app, it is used like this:

```javascript
import ReactDOMStream from "react-dom-stream/server";

app.get('/', (req, res) => {
	ReactDOMStream.renderToStaticMarkup(<Foo prop={value}/>).pipe(res);
});
```

### Rendering on the client

In previous versions of `react-dom-stream`, you needed to use a special render library to reconnect to server-generated markup. As of version 0.2.0, this is no longer the case. You can now use the normal `ReactDOM.render` method, as you would when using `ReactDOM` to generate server-side markup.

## When should you use `react-dom-stream`?

Currently, `react-dom-stream` offers a tradeoff: for larger pages, it significantly reduces time to first byte while somewhat decreasing time to last byte.

For smaller pages, `react-dom-stream` can be a net negative. String construction in Node is extremely fast, and streams and asynchronicity add an overhead. In my testing, pages smaller than about 50KB have worse TTFB and TTLB using `react-dom-stream`. These pages are not generally a performance bottleneck to begin with, though, and on my mid-2014 2.8GHz MBP, the difference in render time between `react-dom` and `react-dom-stream` is usually less than a millisecond.

For larger pages, the TTFB stays relatively constant as the page gets larger (TTFB hovers around 4ms on my laptop), while the TTLB tends to hover around 15-20% longer than `react-dom`. Using this project gets you faster user perceived performance at the cost of worse TTLB performance.

One other operational challenge that `react-dom-stream` can help is introducing asynchronicity, which can allow requests for small pages to not get completely blocked by executing requests for large pages.

I will try in later releases to reduce the extra overhead in `react-dom-stream` in order to make it less of a tradeoff, although it remains to be seen if that can be achieved.

## Who?

`react-dom-stream` is written by Sasha Aickin ([@xander76](https://twitter.com/xander76)), though let's be real, most of the code is forked from Facebook's React.

## Status

This project is of alpha quality; it has not been used in production yet, and the API is firming up. It does, however, pass all of the automated tests that are currently run on `react-dom` in the main React project plus a dozen or so more that I've written.

This module is forked from Facebook's React project. All extra code and modifications are offered under the Apache 2.0 license.

## Something's wrong!

Please feel free to file any issues at <https://github.com/aickin/react-dom-stream>. Thanks!

## Upgrading from v1.x

There was a major change to the API from version 0.1.x to version 0.2.x, as a result of the discussion in issue #2. The version 0.1.x API still work in v0.2.x, but it will be removed in v0.3.x. To learn more about how to upgrade your client code, please read [CHANGELOG.md](/CHANGELOG.md).

## Wait, where's the code?

Well, this is awkward.

You may have noticed that all of the server-side rendering code is in a directory called `lib` and `dist`, which is not checked in to the `react-dom-stream` repo. That's because most of the interesting code is over at <https://github.com/aickin/react/tree/streaming-render-0.2>, which is a fork of React. Specifically, check out [this commit](https://github.com/aickin/react/commit/9159656c53c0335ec6bd56fc7537231a9abeb5d5) to see most of the interesting changes from React 0.14.


## I'd like to contribute!

Awesome. You are the coolest.

To get a working build running and learn a little about how the project is set up, please read [CONTRIBUTING.md](CONTRIBUTING.md).

If you'd like to send PRs to either repo, please feel free! I'll require a CLA before pulling code to keep rights clean, but we can figure that out when we get there.
