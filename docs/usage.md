---
icon: lucide/book-open
---

# Usage

The `Page` class simulates a browser page for testing.
It runs a real V8 engine (via `deno_core`) with jsdom providing the DOM (backed by a native
Rust node tree, selector engine and CSS engine),
and lets you drive it from Python — load pages, query and interact with DOM elements, fill forms,
click and trigger events.
If the page includes htmx, it runs normally and htmx requests are awaited automatically.

## Creating a page

Every page comes from a `Browser`:

```python
from miniclient.browser import Browser, Page

browser = Browser()
page = Page(browser)
...
page.close()
```

`Page` can also be used as a context manager (`with Page(browser) as page:`) to close it on exit.

`Browser(...)` accepts:

- `httpx_transport` — an `httpx2.AsyncBaseTransport`, useful to test an ASGI/WSGI app in-process
  without an HTTP server (see below).
- `mounts` — a `dict[str, Path]` mapping a URL prefix to a local directory, so `<script>` tags can
  load local files (e.g. htmx itself or a bundle) without going through an HTTP request.
- `preboot` — `True` by default. `False` skips the background boot: each page boots its runtime
  when created, which is slower but holds one isolate less in memory.

The `Browser` boots the next page's runtime in the background, so creating a page per test is
nearly free. Pages are independent (own DOM, cookies, event loop and httpx client). The `Browser`
needs no closing: drop it when you are done. To add helpers, subclass `Page` and create it the
same way. With pytest:

```python
@pytest.fixture(scope="session")
def browser():
    return Browser(httpx_transport=WSGITransport(app=app))

@pytest.fixture
def page(browser):
    with Page(browser) as page:
        yield page
```

### Async

An `AsyncPage` with the same constructor is available for async codebases:

```python
from miniclient.browser import AsyncPage, Browser

async with AsyncPage(Browser()) as page:
    ...
```

## Testing a WSGI/ASGI app in-process

You can test your WSGI/ASGI app directly (Django, Flask, FastAPI, etc)
with no HTTP server or network involved, by passing a `miniclient.wsgi.WSGITransport` to `Browser(...)`.

An example with [nanodjango](https://nanodjango.dev/)
_(and [`htmx.min.js`](https://four.htmx.org/docs#download) in the folder `"path/to/htmx/dist"`)_:

```python
from pathlib import Path

from miniclient.browser import Browser, Page
from miniclient.wsgi import WSGITransport
from nanodjango import Django

app = Django()

@app.route("/")
def index(request):
    return """
    <html>
      <head>
        <script src="http://localhost/static/htmx.min.js"></script>
      </head>
      <body>
        <button hx-get="/hello" hx-target="#result">Say hi</button>
        <div id="result"></div>
      </body>
    </html>
    """

@app.route("/hello")
def hello(request):
    return "Hello from Django!"

browser = Browser(
    httpx_transport=WSGITransport(app=app.wsgi),
    mounts={"http://localhost/static/": Path("path/to/htmx/dist")},
)
with Page(browser) as page:
    page.goto("/")
    page.find("button").click()
    print(page.find("#result").text)  # prints "Hello from Django!"
```

For an ASGI app instead, pass an `httpx2.ASGITransport(app=app.asgi)` — see
[httpx2's documentation](https://httpx2.pydantic.dev/advanced/transports/#asgi-transport).

## Loading external scripts via `mounts`

Serve local files through `mounts` so a `<script>` tag can load them without a real server:

```python
page = Page(Browser(mounts={"http://localhost/ext/": tmp_path}))
page.load('<script src="http://localhost/ext/external-script.js"></script>')
```

As in a browser, a `<script>` inserted through `innerHTML` never runs: load it with `load()`,
`goto()`, or by appending a `document.createElement("script")`.

## Loading pages

Load content either via a real request, or directly as raw HTML:

```python
# Fetch a URL and load the document (real request via httpx_transport/network)
page.goto("http://localhost/page")

# Load raw HTML as the document, no request involved
page.load("<p id='msg'>hello</p>")
```

Each call to `load()` replaces the whole document, `<head>` included, with the given HTML; a
fragment like the one above is wrapped in `<html><body>`. Scripts in it run, so htmx on the
page initializes itself as it would in a browser. Unlike `goto()`, `load()` keeps the same
URL, JavaScript realm and history entry.

A new page starts blank at `http://localhost/`. Its first `goto()` reuses that page's JavaScript
realm instead of booting another, so anything `eval()` defined on `window` before it (globals,
listeners) is still there afterwards; every later `goto()` starts a fresh realm. Cookies and
history carry over either way. Elements found before a later `goto()` stop working: find them
again on the new document.

The current URL is available as `page.url` (on an `AsyncPage`, `await page.url()`):

```python
page.goto("http://localhost/page")
assert page.url == "http://localhost/page"
```

Like Playwright's, `goto()` raises `JavaScriptError` when the request itself fails (connection
refused, unknown host), and returns normally for any HTTP status. The page is then left on an
error document holding a `data-mini-navigation-error` element. A navigation the page starts (a
link, a form submit) never raises: it lands on that same error document.

## Finding elements

Locate elements by CSS selector, returning `Element` wrappers:

```python
el = page.find("#msg")          # the first match, or None
items = page.find_all("li")     # a list of all matches, possibly empty
```

Pass `text` to also filter by contained text (a substring match against `textContent`):

```python
el = page.find("li", text="Buy milk")        # first <li> containing this text, or None
items = page.find_all("li", text="urgent")   # all <li>s containing this text
```

`Element` also exposes `find()`/`find_all()`, scoped to that element instead of the whole
document:

```python
row = page.find("#results li")
label = row.find(".label")                      # searches only within `row`
badges = row.find_all(".badge", text="new")
```

## Reading elements

`Element` exposes the usual ways to read content and attributes:

```python
el.html          # outerHTML — the element's tag plus its content
el.inner_html    # innerHTML — the element's content, without its own tag
el.text          # textContent — all text inside, with tags stripped
el.tag_name      # tagName — upper case for HTML elements, e.g. "DIV"
el.attr("href")  # value of the "href" attribute, or None if absent
el.parent        # the parent Element, or None if there is no parent
```

## Filling inputs

Set an input's value with `fill()` or `type()`. They differ only in the event dispatched after
setting the value:

```python
input = page.find("#input-id")
input.fill("new value")  # then dispatches a bubbling `change` event
input.type("new value")  # then dispatches a bubbling `input` event
```

Pick the one your page listens for: `fill()` for `change` handlers (htmx's default trigger for
inputs, textareas and selects), `type()` for `input` handlers such as `hx-trigger="input"`.
Neither one dispatches key events; use `trigger("keyup")` (see below) for a handler like
`hx-trigger="keyup changed delay:500ms"`.

Both work for `<input>`, `<textarea>` and `<select>` elements, and wait for the page to settle.
For `<select>`, the value only takes effect if it matches an existing `<option>`'s value,
just like in a real browser.

## Clicking and triggering events

Simulate a click, or dispatch any DOM event:

```python
page.find("button").click()
page.find("div").trigger("my-event")  # any DOM event, e.g. for hx-trigger="my-event"
```

`trigger()` takes an optional `event_init` dict, passed to the event's constructor. It defaults
to `{"bubbles": True, "cancelable": True}`; pass it to set other fields, such as `key` on a
keyboard event:

```python
page.find("input").trigger("keyup", {"bubbles": True, "key": "Enter"})
```

The event class follows the event name: `MouseEvent` for `click`, `dblclick` and the `mouse*`
events, `KeyboardEvent` for `keydown`/`keyup`/`keypress`, `FocusEvent` for `focus`/`blur`,
`InputEvent` for `input`, `SubmitEvent` for `submit`, and a plain `Event` for anything else.

Both `click()` and `trigger()` wait for the page to settle (any pending timers/fetches, e.g. from
an htmx request, a plain `fetch()`, or another framework's async work).

## Submitting forms

`request_submit()` submits a `<form>` element:

```python
form = page.find("form")
form.request_submit()
```

If a script (e.g. htmx via `hx-post`, `hx-get`, ...) intercepts the submit, this waits for the
page to settle. If not, it performs the form's native GET/POST navigation and reloads the page.
Clicking a `<button type="submit">` or `<input type="submit">` inside the form works the same
way, through `.click()`.

## Executing JavaScript

For anything not covered by `Page` / `Element`, evaluate JavaScript directly.

With sync `Page`, use `eval()`:

```python
title = page.eval("document.title")
```

The code runs as a top-level script: the result is the value of its last statement, and
`var`/`function` declarations stay on `globalThis` for later calls.

`Element` has its own `eval()`, with `this` bound to that element:

```python
value = page.find("#panel").eval("this.dataset.state")
```

An element's `eval()` runs inside a strict-mode function instead, so its declarations stay
local and assigning to an undeclared name throws.

When the JavaScript returns a promise, `eval()` waits for it and returns its resolved value.
Unlike `click()` and the other interactions, it does not wait for the page to settle, since a
script may own a timer that never ends.

On an `AsyncPage`, `eval()` (on the page and on elements) is a coroutine: `await` it:

```python
async with AsyncPage(browser) as page:
    title = await page.eval("document.title")
    status = await page.eval("fetch('/api/status').then(r => r.json())")
    data = await page.find("#panel").eval("this.loadData()")  # returns a promise
```

### Return values

The result crosses into Python as JSON: strings, numbers, booleans, plain objects (as `dict`)
and arrays (as `list`). Both `undefined` and `null` become `None`. Anything JSON would
silently change raises `RuntimeError` instead of arriving as a wrong value: a function, a
Symbol, a BigInt, `NaN` or `Infinity`, an `undefined` inside an object or array, and any object
that is not a plain object or array (a `Map`, a typed array, a DOM node...). A `Date` arrives as
its ISO string.

To read a DOM node back, return the parts you need (`el.textContent`, `el.id`), or use
`find()`.

## Errors

A JavaScript exception surfaces in Python as `JavaScriptError`, carrying the JS error's
`name`, `message` and `stack`:

```python
from miniclient._miniclient import JavaScriptError

try:
    page.eval("null.foo")
except JavaScriptError as error:
    print(error.name)     # "TypeError"
    print(error.message)  # "Cannot read properties of null (reading 'foo')"
```

It is raised by `eval()` when the script throws or its promise rejects, and by `goto()` when the
request fails (see _Loading pages_). An exception thrown by an event listener during `click()`,
`fill()` or `trigger()` is not raised: as in a browser, it is reported and the dispatch goes on.

Calling a method on a closed page, or on an element found before a later `goto()`, raises
`RuntimeError`.

## Configuration

V8 starts with flags tuned for short-lived pages:
`--no-opt --no-regexp-tier-up --min-semi-space-size=16 --initial-old-space-size=128`.
Set the `MINICLIENT_V8_FLAGS` environment variable, before the first page is created, to replace
them. An empty value runs stock V8, which can be faster for a page with a long hot loop or
heavy regular expressions:

```bash
MINICLIENT_V8_FLAGS="" pytest
```
