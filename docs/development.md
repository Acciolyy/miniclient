---
icon: lucide/download
---
# Development

This page covers everything needed to work on this project locally: installing dependencies,
linting, and running the test suite.

## Prerequisites

- A stable [Rust](https://rustup.rs) toolchain (the crate uses edition 2024)
- [Node.js](https://nodejs.org) and npm (CI uses Node 24): the build bundles jsdom with esbuild
- [uv](https://docs.astral.sh/uv/), which also installs the required Python 3.15
- [just](https://just.systems) for the recipes in the `justfile`
- [cargo-nextest](https://nexte.st) for the Rust tests (`cargo install cargo-nextest`)
- For the crosscheck tests only: Playwright's Chromium (`uv run playwright install chromium`)

## Project setup

Install dependencies and vendor htmx:

```bash
uv sync
npm install
./scripts/setup-vendor-htmx.sh   # clones htmx source into vendor/htmx, required by the tests
```

The script checks out the htmx 4.0.0 commit the tests are pinned to. It clones over SSH
(`git@github.com:`); without a GitHub SSH key, run it with `CI=true` to download the same commit
as a tarball instead.

## Building

The Python package wraps a native extension built by [maturin](https://www.maturin.rs). Its
`build.rs` runs `npm ci`, bundles jsdom and the files in `js/` with esbuild, and bakes them into
a V8 snapshot, so a change to any of them, even to a comment, needs a rebuild:

```bash
just develop   # rebuilds the extension into the venv
```

`just test-py` and the WPT recipes run it for you; plain `pytest` does not.

## Linting

We use [prek](https://prek.j178.dev) for linting and formatting. `uv sync` installs it into the
venv (run it as `uv run prek` unless the venv is active). Its hooks run:

- [ruff](https://docs.astral.sh/ruff/) (lint and format) and [pyrefly](https://pyrefly.org/)
  (type check) on Python
- [oxlint](https://oxc.rs/docs/guide/usage/linter) and oxfmt on JavaScript, plus `tsc` on the
  files under `js/` that opt in with `// @ts-check`
- `cargo fmt` and `cargo clippy` on Rust
- a check that `patches/` still matches `vendor/jsdom`, when that checkout exists

To install the git hooks (so they run automatically on each commit):

```bash
prek install
```

To run all hooks manually (auto-fixes most issues in place):

```bash
prek -a
```

## Testing

Most tests are written in Rust, under `tests/*.rs`; the Python tests cover what goes through
the Python layer.

```bash
just test-rust   # cargo nextest run
just test-py     # rebuilds the extension, then pytest -n4
just test        # both
```

A single Rust test runs with `cargo nextest run <filter>`. The `htmx_vendor` test runs htmx's
own suite, and needs `vendor/htmx` (see _Project setup_).

The Python test suite includes some [hypothesis](https://hypothesis.readthedocs.io/) property-based tests
that crosscheck the behavior of this client against a real browser in many different scenarios.
This part of the tests can be rather slow so it's disabled by default.

To run all tests but the crosscheck:

```bash
pytest
```

To run only the hypothesis crosscheck tests:

```bash
pytest -m "cross"
```

To run the full suite, including the crosscheck tests:

```bash
pytest -m ""
```

Tests can run in parallel by adding the `pytest` option `-n auto`.

### WPT conformance

`just test-wpt` runs 140 web-platform-tests files from `fetch/api` and `cors` inside miniclient
(about 1.5 minutes) and fails when a subtest that passed in `tests/wpt/expected.json` no longer
does. After a change that makes more subtests pass, record them with
`just update-wpt`. The files excluded from `tests/wpt/files.txt` need
XMLHttpRequest, iframes, workers or HTTP/2, which miniclient does not provide. The tests come from
`vendor/jsdom/test/web-platform-tests`, the web-platform-tests submodule of a jsdom checkout at
`vendor/jsdom`, which must be checked out. No setup script fetches it, so without it the WPT
recipes and the Rust `chromium_golden` test cannot run (CI skips the latter).

`just test-wpt-extended` runs a larger set (`files-extended.txt`: the rest of
`fetch/api`, `xhr`, form submission, DOM events, timers, `url` and `eventsource`) against
`expected-extended.json`. A single run takes about 20 minutes, so it is not part of `just test-wpt`.

`just test-wpt-css` runs the CSS files in `files-css.txt` (about 20 seconds) against
`expected-css.json`: every file jsdom runs from `cssom`, `css-cascade`, `css-conditional`,
`css-nesting`, `css-variables`, `css-syntax`, `css-values`, `css-animations` and `mediaqueries`, the
`parsing/` directories of the other CSS modules, and jsdom's `to-upstream/css` tests, served under
`/to-upstream/`. Left out: files jsdom skips as slow or timing out, `selectors` (selector matching),
`cssom-view` and `geometry` (layout), and the few files that hang until the per-file timeout. Every
run names the files that ended with no result (`no result: ...`), since they record no passes just
like a file that fails everything.

## Other recipes

- `just check-js` type-checks the JavaScript under `js/` (files opt in with `// @ts-check`).
- `just notice` rewrites `python/miniclient/MPL_NOTICE.md` after a `Cargo.lock` change; the Rust
  tests fail while it is stale.
