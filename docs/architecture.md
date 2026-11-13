---
icon: lucide/layers
---

# Architecture

## Comparison to other tools

Browser automation tools — [Playwright](https://playwright.dev/), [Selenium](https://www.selenium.dev/), [Puppeteer](https://pptr.dev/) — drive a real browser (Chromium, Firefox, WebKit) over a devtools/wire protocol: real rendering, real JS engine, maximum fidelity, but you pay for browser process startup, IPC round-trips, and often GPU/rendering work your test might not need. This project instead runs actual JavaScript code against a real DOM implementation ([jsdom](https://github.com/jsdom/jsdom)) inside an embedded V8 isolate, in-process — no browser process, no protocol, no rendering. That makes the common case (assert a request/DOM-swap/event loop happened correctly) dramatically faster, at the cost of not being a full browser: there is a real CSS cascade with computed values, but no layout or paint, and behavior can diverge anywhere jsdom or this project's polyfills differ from a real engine (see _Known limitations_).

## How the pieces fit

- **Python facade** (`python/miniclient/`): `Browser`, `Page`, `AsyncPage` and `Element`. The
  synchronous `Page` drives the async native core on its own event loop.
- **Native core** (`src/`, Rust via [pyo3](https://pyo3.rs)): each page owns one V8 isolate on a
  dedicated OS thread, booted from a V8 snapshot built at compile time. Python talks to it over a
  command channel. A `Browser` boots the next blank isolate in the background, so creating a page
  rarely waits.
- **jsdom** (bundled with esbuild into the snapshot) provides the `Window` and the DOM APIs, on
  top of hand-written polyfills for the Node modules it imports (`js/polyfills/`). Deno's
  `deno_web`/`deno_fetch` supply the WHATWG APIs (`URL`, streams, `console`, `Request`/`Response`),
  [styleless-innertext](https://www.npmjs.com/package/styleless-innertext) supplies `innerText`,
  and `DataTransfer` comes from
  [@testing-library/user-event](https://github.com/testing-library/user-event).
- **Native DOM and CSS**: the node tree, HTML parsing (html5ever) and selector matching live in
  Rust, behind ops jsdom calls into. CSS runs on [stylo](https://github.com/servo/stylo), Servo's
  style engine.
- **Network**: every `fetch()` the page makes (htmx's included) is handled in JavaScript first
  (CORS, cookies, redirects, `mounts`), then each request is handed to Python and sent through
  `httpx2`, which is how an in-process WSGI/ASGI transport gets to answer it.

A navigation (a `goto()`, a link, a form submit, `location.href = ...`) boots a fresh isolate and
carries cookies and session history into it, like a new document in a real browser tab. The
one exception is a page's first `goto()`, which navigates its still-blank isolate in place.

## How this was made

### Test Methodology

Correctness is checked at four layers:

- **Rust test suite** — regression tests cover the native DOM and selectors, CSS, fetch,
  navigation, the local patches applied to jsdom (via `patch-package`) and the Node polyfills
  it depends on.
- **htmx compat** — we test specifically for htmx's correct functioning in this environment,
  and further guarantee the correctness of the setup as a whole.
- **web-platform-tests** — the browsers' own conformance suite, run from jsdom's checkout: a
  fetch/CORS set, an extended set and a CSS/CSSOM set (23,335 of 26,717 subtests pass as of
  0.3.0). Each run fails on any regression from its recorded baseline.
- **Crosscheck against a real browser** — stateful fuzzing exercises randomly generated pages and interactions, checking that DOM and request/response behave the same as in an actual browser.

#### htmx compat

htmx's own original test suite — including its extensions — runs directly inside this runtime, pinned to htmx 4.0.0: a Rust test (`tests/htmx_vendor.rs`) per directory of the suite runs its Mocha-style files with chai, with a small explicit skip-list for things that can't be true headlessly (e.g. scroll position) or that race a short mock timer. This anchors behavior to the upstream's own definition of correctness.

#### Hypothesis crosscheck against a real browser

We use [Hypothesis](https://hypothesis.readthedocs.io) for property based testing.

Hypothesis drives stateful fuzzing that goes further than any hand-written tests. It generates random HTML — form controls, htmx-attributed elements, whole "rich pages" mixing both, wired to a test app — and draws random sequences of interactions (filling inputs, clicking, dispatching events) against those generated pages. After every step, a `CrossCheck` harness replays the same interaction against both this project's `Page` (V8/jsdom) and a real Chromium page via Playwright, asserting identical DOM snapshots and HTTP request/response parameters. Any divergence, however deep in a randomly generated interaction sequence, fails the test and Hypothesis shrinks it to a minimal reproduction. This process catches subtle DOM/protocol fidelity gaps between jsdom-in-V8 and a real browser that no simple test suite would think to check.

## Known limitations

**No real Node.js.** The JavaScript code runs inside a V8 Isolate, not Node.
There's no node_modules resolution, no require(), no _package.json_ lookup and no module loader: a dynamic `import()` fails with "Module loading is not supported". In practice this means arbitrary npm packages that depend on the Node APIs won't run as-is.

But notice that _jsdom_ itself is a Node package and depends on many parts of the Node API.
It works because this project's esbuild polyfill resolver (`build-jsdom-bundle.mjs`) maps `node:*` specifiers to _polyfills_ (that we made) covering the Node API surface that it touches, baking the result into the V8 snapshot at build time.
Follow the same approach — a resolver plus hand-written polyfills — for any other Node-dependent package you need to run.

A [Vite](https://vite.dev/) production bundle sidesteps this almost entirely: bundling resolves the whole import graph at build time, so nothing needs runtime `require()` or bare-specifier resolution — a standard browser-targeted build just works. The exception is Node-targeted output (SSR, `target: 'node'`) that still calls real `fs`/`child_process`/`os` at runtime — bundling flattens imports, not the runtime APIs, so that code hits the same missing-Node-API wall as unbundled code.

**`<script type="module">` runs as a classic script, not a real ES module.** jsdom fetches and
executes it (`load`/`error` fire normally, same as `async`/`defer`), but there is no module
loader: static `import`/`export` statements are a `SyntaxError` (reported to the console, script
execution just stops there — it still counts as "loaded"). A Vite production bundle sidesteps
this the same way it sidesteps `require()` above: bundling resolves the whole import graph at
build time, so the emitted `<script type="module">` has no `import`/`export` left to resolve.

**`XMLHttpRequest` is not supported.** The constructor exists for feature-detection, but
every method that would actually do I/O (`open`, `send`, `setRequestHeader`, etc.) throws
synchronously. Use `fetch()` for HTTP from page code — it routes through this runtime's
`httpx`-backed client.

**No layout.** Nothing is laid out or painted, so geometry reads as zero (`offsetWidth`,
`getBoundingClientRect()`, `scrollHeight`) and scrolling does nothing. htmx's `scroll:` and
`show:` swap modifiers run without visible effect.

**Observers and media queries.** `IntersectionObserver` exists but never calls its callback, so
htmx's `hx-trigger="intersect"` and `revealed` never fire; trigger the request another way in
tests. `window.matchMedia` and `ResizeObserver` are not defined.

**Navigated documents are decoded as UTF-8.** A document loaded by a navigation is decoded as
UTF-8 whatever its `Content-Type` charset or `<meta charset>` says, so a page served as
`windows-1252`/`latin1` shows wrong accented characters. `fetch()` responses read with
`response.text()` are UTF-8 as well, as the Fetch standard specifies.

**Reduced hardening against malicious page JS.** This project disables one of V8's memory-safety
defenses (Memory Protection Keys) process-wide, to work around a V8 threading bug. This makes V8
somewhat easier to exploit if it ever mishandles memory while running untrusted JavaScript. If
you're testing your own pages or trusted fixtures, this doesn't matter. If you'd point this
project at page content from an untrusted or adversarial source, treat it the same as running any
other JS engine without that protection, and don't rely on it as a security boundary.

**One window per runtime.** This project's own `Runtime` wraps a single `deno_core::JsRuntime` with one V8 `globalThis`, so there's no true per-window isolation — this only matters if the project ever needs to support multiple simultaneous `Window` instances inside one `Runtime`, which isn't possible today. A real navigation (`Page.goto()`, a link click, `location.href =`, a form submit) does get a fresh isolate and a clean `globalThis`, same as a real browser tab — except a page's first `goto()`, which reuses its blank realm, so globals `eval()` defined before it survive. `Page.load()`/`documentWrite` are the one exception: they inject HTML directly with no URL fetch, so they replace the document in the current isolate instead — globals not explicitly scoped to the page (e.g. set by test setup code itself, not by loaded page content) can leak across those.
