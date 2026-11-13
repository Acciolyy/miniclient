---
icon: lucide/zap
---

# miniclient

A minimal **python http client that runs real JS/DOM**, without a browser.<br>
Meant to be used as a lightweight test client to simulate browser interactions.<br>
Embeds a V8 Runtime, DOM, and is designed to run [htmx](https://htmx.org) especially well.

_This project is under active development.
The API is experimental and might change.
We've got a lot of tests but things might break._

## Main Components

- **[httpx2](https://httpx2.pydantic.dev/)** for a fully featured HTTP client, with special support
  for testing WSGI/ASGI apps (Django, Flask, FastAPI).
- **[V8](https://v8.dev/)** via **[deno_core](https://github.com/denoland/deno_core)** for the JavaScript runtime. No Node.js. No browser.
  Deno's own web platform implementations supply `URL`, `TextEncoder`, streams, `console`,
  `performance`, `AbortController` and the Fetch API classes.
- **[jsdom](https://github.com/jsdom/jsdom)** for the DOM and HTML APIs, with the hot paths
  moved to Rust: the node tree is parsed by [html5ever](https://github.com/servo/html5ever) and
  selectors are matched by [selectors](https://crates.io/crates/selectors).
- **[stylo](https://github.com/servo/stylo)**, Servo's style engine, for CSS: the cascade,
  `getComputedStyle`, `element.style` and the CSSOM.

## htmx

- **[htmx](https://htmx.org)** integration is thoroughly tested.
  The complete htmx test suite passes, including extensions, save a short skip-list of tests
  that need layout or race a short mock timer, and most of the design was done to support it.

  _We currently support only htmx version 4 (the suite runs against 4.0.0)_

## Why?

Testing against a real browser feels too slow. And mostly unnecessary.
