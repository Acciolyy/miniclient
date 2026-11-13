---
icon: lucide/rocket
---

# Getting Started

## Installation

```bash
uv add miniclient
```

### Requirements

miniclient needs Python 3.15 or later. Early 3.15 pre-releases install it but fail to import
the extension (`undefined symbol`); upgrade to a 3.15 release. Prebuilt wheels are published
for:

- Linux x86_64 (glibc 2.28 or later)
- macOS on Apple Silicon (arm64)

The wheel already contains V8, jsdom and everything else it runs: no browser, Node.js or driver
to install. On any other platform (Windows, Linux on ARM, macOS on Intel), installing falls back
to building from source, which needs a Rust toolchain, Node.js and network access.

## Quick Tour

The API simulates a browser's user interaction with the `Page` class:

```python
from miniclient.browser import Browser, Page

with Page(Browser()) as page:
    page.goto("http://localhost:8000/")
    page.find("#load-more").click()
    print(page.find("#results").text)
```

Filling in a form and submitting it works the same way, through `fill()` and `request_submit()`:

```python
from miniclient.browser import Browser, Page

with Page(Browser()) as page:
    page.goto("http://localhost:8000/signup")
    page.find("input[name=name]").fill("Ada")
    page.find("input[name=email]").fill("ada@example.com")
    page.find("form").request_submit()
    print(page.find("#result").text)
```

For anything not covered by `Page` / `Element`, you can run arbitrary JavaScript directly
through `eval()`:

```python
from miniclient.browser import Browser, Page

with Page(Browser()) as page:
    page.load("<h1 id='greeting'>Hello</h1>")
    print(page.eval("document.getElementById('greeting').textContent"))
    page.eval("document.getElementById('greeting').textContent = 'Hi!'")
    print(page.find("#greeting").text)
```

## Async usage

An `AsyncPage` equivalent is available for async codebases:

```python
from miniclient.browser import AsyncPage, Browser

async with AsyncPage(Browser()) as page:
    await page.goto("http://localhost:8000/")
    await page.find("#load-more").click()
    print(page.find("#results").text)
```
