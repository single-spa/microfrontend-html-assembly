# microfrontend-html-assembly
Specification for a server assembling an HTML file from one or more microfrontends

## Specification Status

Early Draft, expected to drastically change. We are looking for interested parties to collaborate with on this, and are open to catering to the needs of specific open source communities who might use it.

## Overview

When responding to a browser's HTTP request, a web server (the "root server") must assemble an HTML file. Doing so becomes complex when the HTML is generated dynamically by multiple sources (or "microfrontends"). There is only one HTTP response that is sent to the browser, but each microfrontend may want to influence some or all of the following:

1. HTTP response headers, including Status, Set-Cookie, and more.
2. HTML head content, including `<title>`, `<meta>`, `<link>`, and `<script>` elements.
3. HTML body content, including order of their content in the HTML, in relationship to other microfrontends.

This specification provides a standardized message format between the web server and the microfrontends that allows the microfrontends to accomplish the above. The message format in this specification is guided by the following principles:

1. Fully support streaming content to the browser over time, instead of sending the full HTML file at once.
2. Decentralize logic when sensible, by moving it to the microfrontends instead of the root server.
3. Usable by both single page applications **and** traditional web applications. Not specific to React, Vue, Angular, etc, although UI frameworks may be used as part of generating the HTML content.

## Specification Scope

This specification intentionally does not cover the following topics:

- Determining which microfrontends are involved in assembling the HTML file. This is left to the implementor. For single-spa users, `checkActivityFunctions()` is encouraged.
- Transportation of messages. Although HTTP and JSON are familiar and encouraged as a default, they are not required. This specification can be used in any networked or shared-memory implementation of microfrontends. Its concern is with how the message is structured and streamed, but not with what technologies are used to accomplish that.
- Programming language. Although NodeJS is natural for many microfrontend use cases, it is not required.

## Streaming

For best performance, microfrontend-html-assembly strongly encourages streaming content in the following contexts:

1. Root server to browser. For example, the root server may send the `<head>` content to the browser, including `<script>` and `<link>` resources, before it has assembled the `<body>`. Doing this parallelizes work on the server and browser, since the browser can start fetching resources and displaying content while it waits for the remainder of the content to be sent by the root server.
2. Microfrontend to root server. For example, the microfrontend may send its `<head>` content first, before it is ready to send its `<body>` content.

Streaming over HTTP can be accomplished through Event Streams, which separate messages by a newline. Event streams are implemented with an HTTP response header of `content-type: text/event-stream; application/json; encoding-utf-8;`.

## Render passes

A "render pass" is a programmatic task that generates part of the HTML page. Each render pass results in content being streamed from the root server to the browser. As such, render passes are sequential - the first pass must be finalized and sent to the browser before content from the second pass is sent to the browser.

When communication between root server and microfrontends is done over the network, a single network connection is used instead of one connection per for each render pass.

By default, there are three render passes:

1. HTTP response headers
2. HTML `<head>` content
3. HTML `<body>` content

Each microfrontend participates in zero or more render passes. Render passes may be extended to allow for "render steps" that occur within a single render pass. Render steps are offered as a way to extend render phases for advanced use cases.

## Message format

A "message" is the content sent from microfrontend to root server for a single render pass. Each microfrontend sends multiple messages - one per render pass that it wishes to participate in. Messages may be sent in any json-like format, but must be structured as following:

```js
{
  "renderPass": {
    // the renderPass name must be one of the following: httpHeaders, head, body
    "name": "httpHeaders",
    // "step" is optional, and can be omitted by default. See "render steps" above
    "step": "customString"
  },
  "content": {
    /* The content has a specific shape that is specific to the renderPass (see below)
     * The content property must not be altered or added to. Extensibility can be achieved
     * through the "extra" property
     */
  },
  "extra": {
    // This section is left undefined and is intended for add-ons and extensions to this spec.
  }
}
```

### httpHeaders

All messages for the `httpHeaders` render pass must have a content property that matches the following:

```js
{
  "renderPass": {
    "name": "httpHeaders"
  },
  "content": {
    // All headers are optional. The root server is responsible for providing defaults.
    "headers": [
      "Status: 200",
      "Set-Cookie: id=a3fWa; Max-Age=2592000"
    ]
  }
}
```

### head

All messages for the `head` render pass must have a content property that matches the following:

```js
{
  "renderPass": {
    "name": "head"
  },
  "content": {
    "doctype": "<!DOCTYPE html>",
    "html": "<html lang=en>",
    "head": [
      "<title>Page title</title>",
      "<meta charset=UTF-8>",
      "<meta name=viewport content=width=device-width,initial-scale=1.0>" 
    } 
  }
}
```

### body

All messages for the `head` render pass must have a content property that matches the following:

```js
{
  "renderPass": {
    "name": "body"
  },
  "content": {
    "body": [
      "<div>Some content</div>",
      "<script src=/thing.js>"
    } 
  }
}
```

## Handling duplicates

It is possible for separate microfrontends to send conflicting information to the root-server. For example, one microfrontend may specify an HTTP status of 200 while another specifies 302. Or one may specify `<title>Article 1</title>` while another specifies `<title>News Company.com</title>`.

In such cases, it is up to the root server to determine which to prioritize or whether to throw an error. (More work on this to come later)

## To-dos

- Flesh out duplicate resolution.
- Handle ordering / layout.
