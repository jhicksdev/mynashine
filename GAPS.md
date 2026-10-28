# Myna gaps found while building MynaShine

Environment: `myna-lang` **0.4.2** (language 0.4), CPython **3.12.3**, Linux.
MynaShine did not modify the Myna checkout. The entries below are reproducible against
the released toolchain and are written for the Myna maintainers.

| Gap | Area | Severity | Status |
| :-- | :-- | :-- | :-- |
| G1. `\f` / `\v` escapes compile to literal backslash sequences | lexer / strings | minor (ergonomic) | open |
| G2. JS target rejects relative imports in pure modules | target / bundling | ergonomic (blocks JS port) | open, documented limitation |
| G3. Entry-root relative imports cannot reach a sibling `src/` | module loader | painful (test layout) | open, symlink workaround |

## G1. `\f` and `\v` escapes compile to literal backslash sequences

**Severity:** minor (ergonomic; silent behaviour change for Python-aware authors)

**Reproduction**

```myna
@target(py)
@main {
  s = "a\fb\vc"
  p(#s)
  p([ch | ch <- s])
}
```

```bash
python3 -m myna run escapes.myna
```

**Expected** (following Python string conventions):

```text
5
['a', '\x0c', 'b', '\x0b', 'c']
```

**Actual**

```text
7
['a', '\\', 'f', 'b', '\\', 'v', 'c']
```

The compiled Python contains `s = 'a\\fb\\vc'`; `\f` and `\v` were not decoded. The same
applies to other escapes outside Myna's documented set (`\n`, `\t`, `\r`, `\\`, quotes).
`SPEC.md` only promises the documented escapes, so this is arguably by design rather than
a defect — but it is an easy trap when porting Python lexers that routinely use `\f`/`\v`
for whitespace classes.

**Workaround in MynaShine:** the scanners treat only space, tab, CR and newline as
whitespace. A literal form-feed or vertical-tab character in the input is still emitted
as plain text, so the lossless invariant holds; it simply is not grouped as whitespace.
The initial implementation used `" \t\r\f\v"` and silently classified the letters `f` and
`v` as whitespace, which caused Python function names such as `f` to be swallowed by a
whitespace token. The test suite now covers this indirectly through every corpus file and
malformed fragment.

**Suggestion:** either decode `\f`, `\v`, `\a`, `\b`, `\0`, `\xNN` in string literals, or
raise a lexer error for unknown escapes instead of silently keeping the backslash. A
diagnostic would have surfaced this immediately.

## G2. JavaScript target rejects relative imports in pure modules

**Severity:** ergonomic — blocks the “compile the pure core to JS” future path, not v0.1
functionality

**Reproduction**

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

**Expected:** because the file declares `@target(py, js)` and only uses portable
constructs, it could compile to a single JS module (with imports resolved or bundled).

**Actual**

```text
myna: relative imports are not supported by the JavaScript target;
use an absolute specifier or compile for Python
```

`myna targets src/myna_lexer.myna` reports `js: unsupported — relative imports are not
supported by the JavaScript target`.

This is consistent with the documented “JS target is single-file” limitation, and
`myna bundle` is Python-only. The core modules (`util`, `token`, `myna_lexer`, all language
scanners, `highlight`, `theme`) are nevertheless written without Python-only APIs and
declare `@target(py, js)`; only the import graph blocks standalone JS generation.

**Workaround in MynaShine:** v0.1 ships Python-target CLI/filesystem entrypoints and the
pure modules are kept portable for a future JS bundler. A generated single-file Python
bundle can be produced with `myna bundle`; generated artifacts are not committed.

**Suggestion:** a `--bundle`/`--single-file` mode for `myna compile -t js`, or support for
relative imports in the JS target, would unblock reuse of lexer code in browser-side
tooling without a second source tree.

## G3. Entry-root relative imports cannot reach a sibling `src/`

**Severity:** painful for the recommended `tests/` layout

**Reproduction**

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

**Expected:** `<.src.highlight` resolves to `../src/highlight.myna`, the sibling source
tree, because the project root contains `src/` and `tests/`.

**Actual**

```text
ModuleNotFoundError: No module named '_myna_run_xxxxxxxx.src'
```

The synthetic package root is the **entry file's directory** (`tests/`), so `<.src...`
means `tests/src/...`. Parent-relative imports such as `<..src...` are not available from
the top-level package, and there is no CLI flag to set the module root to the repository
root while keeping the entrypoint in `tests/`.

**Workaround in MynaShine:** commit a symlink:

```bash
ln -s ../src tests/src
```

Then `<.src.highlight` resolves through `tests/src` to the real modules, and imports
inside those modules (for example `<.util` in `src/token.myna`) resolve within the
`src` subpackage. The symlink is committed so CI and fresh clones work unchanged.

**Suggestion:** support a `--module-root`/`MYNA_MODULE_ROOT` option, infer the project
root from a `site.toml`/`pyproject.toml`, or permit `..` to climb from the synthetic
top-level package. Any of these would remove the need for layout-specific symlinks.

---

No compiler crashes, incorrect tokenisation of valid Myna source, or code-generation
failures were encountered beyond the items above. The lossless corpus (all `.myna` files
in MynaShine, the Myna examples, and MynaPress `src/`; 52 files at time of writing)
highlighted without exceptions.
