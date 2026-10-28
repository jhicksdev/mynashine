# Initial prompt: build MynaShine

You are starting a new project from scratch in `/home/joseph/Projects/mynashine`.
Read this whole file before writing any code.

## Mission

Build **MynaShine**, a small, dependency-free syntax highlighter for the Myna programming
language (plus the common companion languages used in MynaPress documentation and posts),
written **entirely in Myna**.

MynaShine's primary purpose in v0.1 is **HTML + CSS syntax highlighting for fenced code
blocks in MynaPress** (`/home/joseph/Projects/mynapress`), not text-editor plugins (such
as VS Code TextMate grammars, Tree-sitter, or LSP semantic tokens — those are explicit
non-goals for v0.1).

MynaShine exists for three reasons, in this order:

1. **Make Myna code readable and visually distinct on the web**, especially inside
   MynaPress sites where Myna's dense sigils (`?`, `%`, `^`, `!`, `<`, `@`, `$$`) benefit
   enormously from semantic color distinguishing statement-prefix keywords from infix
   operators.
2. **Provide a drop-in module and CLI** that MynaPress (or any static site generator) can
   call or import to highlight `<pre><code class="language-...">` blocks at build time
   with zero JavaScript required in the browser.
3. **Continue dogfooding Myna** on a lexer/scanner and HTML transformation workload,
   recording any language friction in `GAPS.md`.

It will be published on GitHub (`jhicksdev/mynashine`) as an open-source project.

## Read the existing projects first — this is mandatory

Two sibling projects on this machine are the sources of truth:

1. **Myna (`/home/joseph/Projects/myna`)** — the language specification, compiler, and
   reference lexer. Do not invent syntax, and do not guess at Myna's lexical rules.
2. **MynaPress (`/home/joseph/Projects/mynapress`)** — the static site generator that
   MynaShine is built to serve.

Before writing your first `.myna` file, read:

| Path | What it gives you |
| :--- | :--- |
| `/home/joseph/Projects/myna/SPEC.md` | Full Myna language specification (v0.4, current release `0.4.2`) |
| `/home/joseph/Projects/myna/skills/myna/SKILL.md` | Compact "Rosetta Stone" for writing idiomatic Myna |
| `/home/joseph/Projects/myna/myna/lexer.py` | Reference tokenizer for Myna — check how strings, f-strings (`` `...{expr}...` ``), raw strings (`_"..."`), bytes (`0b"..."`), comments (`#`), and multi-character sigils are lexed |
| `/home/joseph/Projects/myna/myna/parser.py` | Statement-start vs. expression-infix disambiguation rules |
| `/home/joseph/Projects/mynapress/src/markdown.myna` | How MynaPress currently parses fenced code blocks and emits `<pre><code class="language-...">` |
| `/home/joseph/Projects/mynapress/assets/style.css` | MynaPress's light/dark CSS custom property palette (`--bg`, `--fg`, `--muted`, `--accent`, `--code-bg`) |
| `/home/joseph/Projects/mynapress/GAPS.md` | Known Myna gaps and what was already fixed in `myna-lang` 0.4.x |

### Environment note

Use the local Myna installation or editable checkout at `/home/joseph/Projects/myna`:

```bash
python3 -m pip install -e /home/joseph/Projects/myna
# or run with:
PYTHONPATH=/home/joseph/Projects/myna python3 -m myna run ...
```

Useful commands:

```bash
python3 -m myna run src/main.myna ...              # execute via Python
python3 -m myna bundle src/main.myna -o dist/mynashine.py
python3 -m myna stats src/
```

## Hard constraints

- **All application and test code is `.myna`.** No Python modules implementing product
  behaviour. A shell wrapper (`bin/mynashine`) and CI workflow files are the only
  non-Myna code allowed.
- **Zero runtime dependencies** beyond `myna-lang` (`>=0.4.2`) and the Python standard
  library.
- **Target Python 3.11+ (`@target(py)`)** for CLI/filesystem entrypoints, while keeping
  the core tokenizer/highlighter modules (`lexer.myna`, `highlight.myna`, `theme.myna`)
  pure and portable (`@target(py, js)`) if feasible so they can also compile to JS later!
  (If `@target(py, js)` hits a blocker in a pure module, document it in `GAPS.md` and
  fall back to `@target(py)`.)
- **Never hand-edit or commit generated `.py` or `.js` files.**
- **Do not modify `/home/joseph/Projects/myna`** while building MynaShine. Document any
  compiler or language issues in `GAPS.md`.
- **Fault-tolerant highlighting:** A syntax highlighter must **never crash or fail a
  MynaPress build** on incomplete, malformed, or pseudo-code snippets. Unrecognized
  characters or unterminated tokens must degrade gracefully to plain escaped text tokens.
- **Lossless round-trip invariant:** For every input string `code`, stripping all HTML
  tags and unescaping HTML entities from `highlight_html(code, lang)` **must reproduce
  `code` byte-for-byte** (including all whitespace, newlines, and comments).

## What makes highlighting Myna unique (and why regex-only fails)

Myna reuses ASCII punctuation for both **statement-level control flow / declarations** and
**expression-level operators**:

| Sigil | Statement-prefix role (highlight as keyword/declaration) | Expression role (highlight as operator/builtin) |
| :--- | :--- | :--- |
| `?` / `:?` / `:` | `if` / `elif` / `else` (`? cond {`, `:? cond {`, `: {`) | Ternary `c ? a : b`, type check `x :? T`, type annotation `:i`, slice `a:b` |
| `??` | `match` statement (`?? subj {`) | Nullish coalescing `a ?? b` |
| `%` / `%~` / `%?` / `%:` | `for`, `async for`, `while`, loop `else` | Modulo `a % b`, comprehension clause `% y <- it2` |
| `^` / `^*` / `^**` / `^!` / `^>` | `return`, `yield`, `yield from`, `break`, `continue` | Bitwise XOR `a ^ b` (`^= x` is `nonlocal x` at statement start) |
| `!` / `!!` / `!:` / `!?` / `!^` / `!&` / `!~&` | `try`, `except`, `finally`, `else`/`assert`, `raise`, `with`, `async with` | Logical NOT `!x`, `!=`, `!===`, `!<-` |
| `<` | `import` statement at statement start (`<os`, `<.util:{f}`) | Less-than comparison `a < b` or class inheritance `$C < B` |
| `~` | `del x` at statement start (`~x`), `~@f` async def | `await x` (`~x`) or bitwise NOT `~\|x` in expressions |
| `@` / `~@` / `*@` / `@/` / `@$` / `@?` / `@=` / `+@` | Function/method/property declarations, `@= x` (`global`), `@target`, `@main` | Builtin shorthands (`@enum`, `@zip`, `@map`, `@flt`, `@srt`, `@rev`, `@sum`, `@min`, `@max`, `@str`, `@int`, `@flt_n`) |
| `$` / `$$` / `$=` / `+$` | Class (`$C`), dataclass (`$$C`), type alias (`$= A = ...`), exported class (`+$C`) | — |
| `.` / `..` | `.x` (`self.x`), `..init()` (`super().__init__()`) | Attribute access `a.b`, range `a..b`, spread |
| `#` | `# comment` (when followed by space, `#`, or EOL; or at statement start) | Length operator `#x` in expressions, `#[dec]` decorator |

Because of this, MynaShine must track lightweight lexical context (such as whether the
scanner is at the **start of a statement** — after a newline, `;`, `{`, or `=>` — and
whether it is inside a type annotation `: Type` or `-> Type`, or inside a backtick
interpolated string `` `...{expr}...` ``) so that:

1. Statement-prefix sigils (`?`, `:?`, `??`, `%`, `%?`, `^`, `!{`, `!!`, `<mod`, etc.) get
   a **control/keyword** token class.
2. Declaration sigils (`@f`, `~@f`, `$C`, `$$C`, `$=`, `#[...]`, `@init`, `@target`,
   `@main`) and the declared identifier immediately following them get **declaration** and
   **function/class name** token classes.
3. Built-in shorthands (`p`, `@enum`, `@str`, etc.) and literal constants (`T`, `F`, `N`,
   `U`) get **builtin** and **constant** token classes.
4. Type shorthands (`i`, `f`, `s`, `b`, `B`, `A`, `N`) are highlighted as **types** *only*
   in type positions (after `:` in parameters/variables, after `->`, after `:?`, after
   `$=`, or inside `[T]` / `{K:V}` type expressions), never when used as ordinary local
   variable names like `i = 0` or `s = "hello"`.
5. Backtick f-strings (`` `hello {name} {val:.2f}` ``) highlight the outer string as a
   string literal, `{` / `}` and format specs `:.2f` as interpolation punctuation/spec,
   and recursively highlight the inner expression `name` / `val` as Myna tokens!

## Product scope for v0.1

### 1. Supported languages

- **`myna`** (aliases: `myna`, `mn`) — full context-aware lexical highlighting as specified
  above.
- **Companion languages used in MynaPress docs & posts** (lightweight, fault-tolerant
  scanners):
  - **`python`** (`py`, `python`, `python3`) — keywords, builtins, decorators, strings
    (including `f"..."`, `r"..."`, triple-quoted), comments, numbers, function/class names.
  - **`javascript`** (`js`, `javascript`, `mjs`) — keywords, strings, template literals,
    comments (`//`, `/* */`), numbers, declarations.
  - **`toml` / `yaml`** (`toml`, `yaml`, `yml`) — keys, strings, booleans, numbers,
    comments, section headers `[section]`, front-matter `---`.
  - **`bash` / `shell`** (`sh`, `bash`, `shell`, `console`) — prompts (`$`), commands,
    flags (`--config`), strings, comments, env vars.
  - **`html` / `markdown` / `myna-template`** (`html`, `xml`, `md`, `markdown`, `jinja`,
    `template`) — HTML tags, attributes, entities, plus MynaPress template tags
    `{{ ... | filter }}` and `{% ... %}`!
  - **`text` / unknown** (`text`, `plain`, `txt`, or any unrecognized tag) — safe
    HTML-escaped fallback without error.

### 2. Token taxonomy & HTML output

Use concise, semantic CSS classes with a `hl-` prefix so generated HTML stays compact and
readable:

| Token category | CSS class | Examples in Myna |
| :--- | :--- | :--- |
| Keyword / Control sigil | `hl-kw` | `?`, `:?`, `:`, `??`, `%`, `%?`, `^`, `^!`, `^>`, `<-` (in loop), `!{`, `!!`, `!^`, `!&`, `=>` |
| Declaration sigil | `hl-decl` | `@`, `~@`, `*@`, `@/`, `@$`, `@?`, `@=`, `+@`, `$`, `$$`, `$=`, `+$` |
| Import / Module | `hl-imp` | `<`, `<.`, `@target`, `@main` |
| Decorator | `hl-dec` | `#[...]` |
| Function / Method name | `hl-fn` | `render_blocks` in `@render_blocks(...)`, `@init` |
| Class / Type name | `hl-cls` | `Page` in `$$Page(...)` or `$Server` |
| Type shorthand / annotation | `hl-type` | `i`, `f`, `s`, `b`, `B`, `A`, `N`, `->` in type context |
| Builtin / Shorthand | `hl-bi` | `p`, `@enum`, `@zip`, `@map`, `@flt`, `@srt`, `@rev`, `@sum`, `@min`, `@max`, `@str`, `@int`, `@flt_n` |
| Self / Super | `hl-self` | leading `.attr` dot/ident, `..init` |
| Boolean / Null constant | `hl-const` | `T`, `F`, `N`, `U` |
| Number literal | `hl-num` | `42`, `3.14`, `0xff` |
| String / Raw / Bytes | `hl-str` | `"..."`, `'...'`, `_"..."`, `0b"..."`, `` `...` `` |
| String interpolation | `hl-interp` | `{` and `}` and `:spec` inside `` `...{expr}...` `` |
| Operator | `hl-op` | `\|>`, `:=`, `&&`, `\|\|`, `===`, `!==`, `..`, `:?`, `#` (len), `!` (not) |
| Punctuation / Delimiter | `hl-punc` | `{`, `}`, `(`, `)`, `[`, `]`, `,`, `;`, `:` |
| Comment | `hl-com` | `# comment` |
| Identifier / Plain | *(no span needed, or merged)* | local variables, whitespace |

- **Span merging & compactness:** Adjacent tokens of the same class (or plain text/whitespace)
  must not produce redundant wrapper tags. Plain identifiers and whitespace should be emitted
  as bare HTML-escaped text (no `<span>` wrapper) to keep output HTML small.
- **Optional line wrappers / line numbers:** Support an option `line_numbers:b = F` that
  wraps lines in `<span class="hl-line" data-line="1">...</span>` for code blocks that
  want line numbering.

### 3. Themes & CSS generation

Provide built-in CSS generation (`mynashine css`) designed to harmonize with MynaPress out
of the box:

- **`mynapress` (default theme):** Uses CSS custom properties (`--hl-kw`, `--hl-str`, etc.)
  with automatic light and `@media (prefers-color-scheme: dark)` palettes tuned for
  MynaPress's warm paper (`#fbfaf7`) and warm charcoal (`#171715`) backgrounds, passing
  WCAG AA contrast in both modes.
- Additional named palettes (`light`, `dark`, `high-contrast`) selectable via CLI flag.

### 4. Integration with MynaPress (two first-class workflows)

Because MynaPress is a sibling project, MynaShine must support **both** ways of working
with MynaPress:

1. **Direct HTML post-processor (`mynashine html` / `mynashine site`):**
   Can take an already-rendered HTML file or a MynaPress `build/` directory, find every
   `<pre><code class="language-...">...</code></pre>`, unescape its inner text, highlight
   it into `<pre class="hl"><code class="language-...">...</code></pre>`, and optionally
   inject or append the MynaShine CSS into `assets/style.css`. This works on *any*
   MynaPress site immediately without touching MynaPress's source!
2. **Direct Myna module API (`src/highlight.myna`):**
   Clean public functions:
   - `highlight_code(code:s, lang:s="myna", line_numbers:b=F)->s` (returns inner HTML)
   - `highlight_block(code:s, lang:s="myna", line_numbers:b=F)->s` (returns full `<pre class="hl"><code...>`)
   - `highlight_html_document(html:s)->s` (highlights all `<pre><code...>` blocks in an HTML string)
   - `default_css(theme:s="mynapress")->s`
   Plus a single-file bundle (`dist/mynashine.myna` or documented copy-in module) so
   MynaPress can vendor or import it directly into `mynapress/src/markdown.myna` when
   desired.

### 5. CLI

```bash
myna run src/main.myna code <file> [--lang myna] [--lines] [--wrap]
myna run src/main.myna html <input.html> [-o output.html]
myna run src/main.myna site <build_dir> [--inject-css]
myna run src/main.myna css [--theme mynapress] [-o style.css]
myna run src/main.myna demo [-o build/index.html]
```

- `code` — highlight a source file (or `-` for stdin) and print HTML to stdout.
- `html` — highlight all `<pre><code>` blocks in an HTML file.
- `site` — walk a built site directory (such as `../mynapress/build`), highlight all
  `<pre><code>` blocks in `.html` files in-place, and with `--inject-css` append the
  MynaShine stylesheet rules to `assets/style.css` (idempotently).
- `css` — emit the stylesheet for the chosen theme.
- `demo` — generate a self-contained showcase HTML page demonstrating Myna syntax
  highlighting across real snippets from `myna/examples/` and `mynapress/src/`, plus
  companion languages, in both light and dark modes.

## Suggested architecture

```text
src/
  main.myna          # CLI entrypoint
  cli.myna           # argument parsing, help, subcommand dispatch
  token.myna         # Token dataclass, TokenKind constants, span emission & lossless check
  myna_lexer.myna    # Context-aware Myna syntax highlighter (the flagship scanner)
  lang_py.myna       # Python scanner
  lang_js.myna       # JavaScript scanner
  lang_config.myna   # TOML / YAML front-matter scanner
  lang_shell.myna    # Bash / shell scanner
  lang_markup.myna   # HTML / Markdown / MynaPress template ({{ }}, {% %}) scanner
  highlight.myna     # Public API: dispatch by lang, HTML block rendering, HTML doc post-processing
  theme.myna         # CSS custom properties & light/dark themes tuned for MynaPress
  site_proc.myna     # Directory walker & idempotent CSS injector for MynaPress build/ dirs
  demo.myna          # Showcase HTML generator
  util.myna          # HTML escape/unescape, character classification helpers
tests/
  run_tests.myna     # Test runner
  test_myna_lexer.myna
  test_companions.myna
  test_html_proc.myna
  test_lossless.myna
```

## Tests — written in Myna

- Write a test runner in `tests/run_tests.myna` that exits non-zero on failure.
- **Disambiguation tests (`test_myna_lexer.myna`):** Explicitly test every dual-purpose
  Myna sigil:
  - `?` as `if` vs. ternary `c ? a : b`
  - `:?` as `elif` vs. `isinstance` check `x :? [s]`
  - `??` as `match` vs. nullish coalescing `a ?? b`
  - `%` as `for` loop vs. modulo `x % 2` vs. comprehension `% y <- ys`
  - `^` as `return` vs. XOR `a ^ b` vs. `^= x` (`nonlocal`)
  - `!` as `try`/`except`/`raise`/`with` vs. `!x` (`not`)
  - `<` as `import` (`<re`, `<.util:{html_escape}`) vs. comparison `i < n` vs. inheritance `$C < B`
  - `#` as comment vs. `#x` length operator vs. `#[dec]` decorator
  - Type shorthands `i`, `s`, `b`, `f`, `A`, `N` in `@fn(i:i, s:s)->b` (the parameter
    names `i` and `s` must be plain identifiers; `:i`, `:s`, `->b` must be `hl-type`!)
  - Nested expressions inside backtick strings `` `result: {x + 1:.2f}` ``
- **Lossless round-trip corpus test (`test_lossless.myna`):** Run `highlight_code` on
  every `.myna` file in `/home/joseph/Projects/myna/examples/` and
  `/home/joseph/Projects/mynapress/src/`, strip `<span...>`/`</span>` and unescape HTML
  entities, and assert exact equality with the original source file.
- **Malformed input resilience test:** Feed unterminated strings, unmatched braces,
  stray sigils, and random ASCII sequences to every language scanner and assert both no
  exception and lossless text preservation.
- **MynaPress integration test (`test_html_proc.myna`):** Render Markdown with fenced
  ```` ```myna ```` blocks, run `highlight_html_document`, and verify the resulting HTML.

## Handling language gaps

Maintain **`GAPS.md`** at the repo root documenting any friction, bugs, or ergonomic
issues encountered in `myna-lang` 0.4.2 while building MynaShine, including a minimal
reproduction, expected vs. actual behavior, workaround used, and severity. If zero new
compiler bugs are hit, record ergonomic observations honestly.

## Definition of done (v0.1)

All of the following must be true and verified before completion:

- [ ] `python3 -m myna run tests/run_tests.myna` passes all unit, disambiguation,
  malformed-input, and lossless corpus tests.
- [ ] Every `.myna` file in `/home/joseph/Projects/mynapress/src/` highlights losslessly.
- [ ] `python3 -m myna run src/main.myna demo -o build/demo.html` produces a polished,
  self-contained interactive/responsive showcase page (with light/dark toggle) featuring
  real MynaPress and Myna code blocks.
- [ ] `python3 -m myna run src/main.myna site` is verified against a build of MynaPress
  (in a temp directory or fixture) and cleanly highlights all fenced code blocks.
- [ ] `README.md` documents installation, CLI commands, MynaPress integration (both the
  `mynashine site` post-processor and direct Myna module import), token classes, theming,
  and architecture.
- [ ] `GAPS.md`, `LICENSE` (MIT), `.gitignore`, `bin/mynashine` wrapper, and GitHub
  Actions CI (`.github/workflows/ci.yml`) are in place.
- [ ] Git repository is initialized on branch `main` with clean commits ready to push
  to GitHub (`jhicksdev/mynashine`).

## Milestones

Work in this order; commit at each green milestone:

1. **M0 spike:** Core `Token` model, lossless strip/unescape verifier, and basic Myna
   scanner on a 20-line snippet.
2. **M1:** Full context-aware Myna lexer (`myna_lexer.myna`) covering statement-prefix vs.
   infix sigils, type positions, and backtick string interpolation + unit tests.
3. **M2:** Companion language scanners (`python`, `js`, `toml`/`yaml`, `shell`,
   `html`/`markdown`/`template`) + resilience tests.
4. **M3:** HTML block/document highlighter, `mynapress` light/dark CSS theme generator,
   and `site` post-processor.
5. **M4:** CLI (`code`, `html`, `site`, `css`, `demo`), `bin/mynashine`, showcase page,
   `README.md`, `GAPS.md`, and CI workflow.
