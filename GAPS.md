# Myna gaps found while building MynaShine

Environment: `myna-lang` **0.4.2** plus the post-0.4.2 fixes on the local Myna `main`
working tree (2026-10-28), CPython **3.12.3**, Linux. The fixes were unreleased at the
time of writing; MynaShine is verified against the fixed checkout and remains usable with
released 0.4.2 for the CLI/module API.

MynaShine did not modify the Myna checkout. All three gaps below are now **fixed
upstream** and re-verified.

| Gap | Area | Severity | Status |
| :-- | :-- | :-- | :-- |
| G1. `\f` / `\v` escapes compile to literal backslash sequences | lexer / strings | minor (ergonomic) | **Fixed** — full Python escape set |
| G2. JS target rejects relative imports in pure modules | target / bundling | ergonomic (blocked JS port) | **Fixed** — multi-file ES modules |
| G3. Entry-root relative imports cannot reach a sibling `src/` | module loader | painful (test layout) | **Fixed** — `--module-root` / `MYNA_MODULE_ROOT` |

## G1. `\f` and `\v` escapes compile to literal backslash sequences — fixed

**Severity:** minor (ergonomic; silent behaviour change for Python-aware authors)

### Original reproduction

```myna
@target(py)
@main {
  s = "a\fb\vc"
  p(#s)
  p([ch | ch <- s])
}
```

Before the fix:

```text
7
['a', '\\', 'f', 'b', '\\', 'v', 'c']
```

The compiled Python contained `s = 'a\\fb\\vc'`; `\f` and `\v` were not decoded, so the
letters `f` and `v` silently became part of the string data. MynaShine's first lexer used
`" \t\r\f\v"` as its whitespace set and consequently treated the *letters* `f` and `v` as
whitespace, swallowing Python function names such as `f` into a whitespace token.

### Fix and verification

Myna now decodes the full Python string escape set (`\a \b \f \v \xNN \uNNNN \UNNNNNNNN
\ooo \N{NAME}` and backslash-newline) and raises a `LexError` for unknown or malformed
escapes instead of silently keeping the backslash. The original reproduction now prints:

```text
5
['a', '\x0c', 'b', '\x0b', 'c']
```

MynaShine deliberately keeps its scanners free of `\f`/`\v` escape literals so the CLI
still runs on released 0.4.2. Actual form-feed/vertical-tab characters pass through as
plain text and remain lossless; the malformed-input corpus includes `"a\fb\vc"` across
every language.

## G2. JavaScript target rejects relative imports in pure modules — fixed

**Severity:** ergonomic — previously blocked compiling the pure core to JS

### Original reproduction

```bash
cd /home/joseph/Projects/mynashine
python3 -m myna compile src/myna_lexer.myna -t js -o /tmp/myna_lexer.js
```

`src/myna_lexer.myna` starts with:

```myna
@target(py, js)
<.token:{tok}
<.util:{is_digit, is_id_start, is_id_cont, is_space, in_text, in_list}
```

Before the fix:

```text
myna: relative imports are not supported by the JavaScript target;
use an absolute specifier or compile for Python
```

`myna targets src/myna_lexer.myna` reported `js: unsupported`.

### Fix and verification

Relative imports now compile to multi-file ES modules (`<.util:{f}` →
`import { f } from "./util.js"`), and `myna run -t js` compiles the reachable graph into
temporary `.mjs` files before running Node. `myna targets src/myna_lexer.myna` now reports:

```text
py       : supported (generation only)
js       : supported (generation only)
```

MynaShine adds `tests/js_smoke.myna`, a `@target(py, js)` smoke test for the portable core:

```bash
python3 -m myna run -t js --module-root . tests/js_smoke.myna
```

It verifies lossless Myna highlighting, keyword/type/interpolation tokens, CSS generation,
and the HTML document processor under Node:

```text
js-lossless=true
js-keyword=true
js-type=true
js-interp=true
js-css=true
js-document=true
js-smoke-ok=true
```

Single-file JavaScript **bundling** is still out of scope (`myna bundle` remains
Python-only); the supported JavaScript unit is a tree of ES modules.

## G3. Entry-root relative imports cannot reach a sibling `src/` — fixed

**Severity:** painful for the recommended `tests/` layout

### Original reproduction

```text
project/
  src/highlight.myna
  tests/run_tests.myna
```

```myna
# tests/run_tests.myna
@target(py)
<.src.highlight:{highlight_code}
@main {
  p(highlight_code("? x { ^1 }"))
}
```

```bash
python3 -m myna run tests/run_tests.myna
```

Before the fix:

```text
ModuleNotFoundError: No module named '_myna_run_xxxxxxxx.src'
```

The synthetic package root was the entry file's directory (`tests/`), so `<.src...` meant
`tests/src/...`. Parent-relative imports were not available from the top-level package, and
there was no way to set the module root without moving the entrypoint.

### Fix and verification

`myna run` and `myna bundle` now accept `--module-root DIR` (or `MYNA_MODULE_ROOT`). The
entry file's relative imports resolve from `DIR`; imported modules continue to resolve
against their own package. MynaShine's layout uses:

```bash
python3 -m myna run --module-root . tests/run_tests.myna
```

- `tests/run_tests.myna` imports siblings as `<.tests.test_...>`.
- Test modules import the source tree as `<..src...>`.
- The `tests/src -> ../src` symlink workaround has been removed.

The full suite passes: **1142 assertions** across the lexer, companion languages, HTML
processing, corpus, and site processor.

---

No compiler crashes, incorrect tokenisation of valid Myna source, or code-generation
failures were encountered beyond the items above. The lossless corpus (all `.myna` files
in MynaShine, the Myna examples, and MynaPress `src/`; 38 files at time of writing)
highlights without exceptions on both the Python and JavaScript targets.
