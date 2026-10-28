# MynaShine

**A small, dependency-free syntax highlighter for [Myna](https://github.com/jhicksdev/myna) and its companion languages — written entirely in Myna.**

MynaShine exists to make Myna readable on the web. It highlights fenced code blocks at
build time and emits compact semantic HTML with **zero JavaScript in the browser**. Its
default theme is tuned for [MynaPress](https://github.com/jhicksdev/mynapress), but it works
on any static site.

```myna
@target(py)
<asyncio
$$Task(id:i, title:s, done:b=F) {
  @summary->s {
    ^ `[{.done ? "x" : " "}] #{.id}: {.title}`
  }
}
```

Myna's dense sigils are the whole point: `?` is both `if` and a ternary operator, `%` is
both `for` and modulo, `^` is both `return` and XOR, and `<` is both `import` and
less-than. MynaShine tracks enough lexical context — statement starts, loop headers, type
positions, leading self references, and interpolation — to colour them correctly without
parsing a full AST.

## What you get

- **Myna-aware**: statement sigils, declarations, type shorthands, builtins, constants,
  self/super references, decorators, imports, and recursive highlighting inside
  `` `...{expr}...` `` template strings.
- **Companion languages**: Python, JavaScript, TOML, YAML, Bash/shell, HTML/XML,
  Markdown, and MynaPress template syntax (`{{ ... | filter }}` and `{% ... %}`).
- **Lossless**: stripping tags and unescaping entities always reproduces the input
  byte-for-byte, including whitespace, newlines, comments, and malformed code.
- **Fault-tolerant**: unterminated strings, stray sigils, unmatched braces, and random
  ASCII degrade to escaped text. Highlighting never crashes a build.
- **Compact output**: plain identifiers and whitespace are bare text; adjacent tokens of
  the same class merge into one span.
- **Four themes**, light/dark support, optional line numbers, and an idempotent
  post-processor for existing site builds.
- **No runtime dependencies** beyond `myna-lang >= 0.4.2` and the Python standard library.

## Requirements

- Python 3.11+
- [`myna-lang`](https://pypi.org/project/myna-lang/) 0.4.2 or newer

```bash
python3 -m pip install "myna-lang>=0.4.2"
```

For local development against a Myna checkout:

```bash
PYTHONPATH=/home/joseph/Projects/myna python3 -m myna run src/main.myna --help
```

## Quick start

The repository ships a thin shell launcher:

```bash
./bin/mynashine code page.myna
./bin/mynashine code page.myna --lang myna --lines
./bin/mynashine code - --lang myna < page.myna
./bin/mynashine css --theme mynapress -o assets/highlight.css
```

Or invoke the entrypoint directly:

```bash
python3 -m myna run src/main.myna code page.myna --wrap
```

### CLI

```text
mynashine code <file|-> [--lang LANG] [--lines] [--wrap]
mynashine html <input.html> [-o output.html]
mynashine site <build_dir> [--inject-css] [--theme THEME]
mynashine css [--theme THEME] [-o style.css]
mynashine demo [-o build/demo.html]
```

| Command | Behaviour |
| :-- | :-- |
| `code` | Highlight a source file (or `-` for stdin) and print inner HTML. `--wrap` emits a full `<pre class="hl"><code class="language-...">` block. `--lines` adds line wrappers. |
| `html` | Highlight every `<pre><code class="language-...">` block in an HTML file. Writes to stdout unless `-o` is given. |
| `site` | Walk a built site (`build/`), highlight `.html` files in place, and optionally inject the stylesheet into `assets/style.css` with `--inject-css`. Idempotent. |
| `css` | Emit a theme stylesheet. Themes: `mynapress` (default), `light`, `dark`, `high-contrast`. |
| `demo` | Generate a self-contained responsive showcase page with real Myna and MynaPress source excerpts. |

Every command exits non-zero on a real I/O error. Highlighting itself is fault-tolerant.

## MynaPress integration

### 1. Post-process an existing build

This is the drop-in path; it requires no changes to MynaPress:

```bash
# Build the site as usual, then:
./bin/mynashine site ../mynapress/build --inject-css
```

MynaShine finds blocks like:

```html
<pre><code class="language-myna">? x { ^1 }</code></pre>
```

and rewrites them to:

```html
<pre class="hl"><code class="language-myna"><span class="hl-kw">?</span> x <span class="hl-punc">{</span> <span class="hl-kw">^</span><span class="hl-num">1</span> <span class="hl-punc">}</span></code></pre>
```

The CSS injection is wrapped in stable markers:

```css
/* mynashine:start */
...
/* mynashine:end */
```

Re-running `site` replaces that block instead of appending another copy, and already
highlighted blocks are skipped. Blocks without a language class are treated as `text`;
unknown languages are safely escaped.

### 2. Import the module directly

Copy `src/` into a project as a `mynashine/` package (or vendor a bundle), then import the
public API:

```myna
@target(py)
<.mynashine.highlight:{highlight_code, highlight_block, highlight_html_document, default_css}

# Inside MynaPress's markdown renderer:
block = highlight_block(code, lang, F)

# Or post-process the final rendered page:
page_html = highlight_html_document(page_html)
stylesheet = default_css("mynapress")
```

Public functions:

```myna
highlight_code(code:s, lang:s="myna", line_numbers:b=F)->s
highlight_block(code:s, lang:s="myna", line_numbers:b=F)->s
highlight_html_document(html:s)->s
default_css(theme:s="mynapress")->s
```

`highlight_code` returns the inner HTML for a `<code>` element. `highlight_block` returns
the complete `<pre class="hl"><code class="language-...">...</code></pre>` wrapper.

### Single-file bundle

`myna bundle` can produce a standalone Python artifact for a vendored module or the CLI:

```bash
python3 -m myna bundle src/main.myna -o dist/mynashine.py
python3 dist/mynashine.py site ../mynapress/build --inject-css
```

Generated `.py`/`.js` files are build artifacts and are not committed.

## Token classes

Output uses concise semantic classes with an `hl-` prefix:

| Category | Class | Examples |
| :-- | :-- | :-- |
| Keyword / control sigil | `hl-kw` | `?`, `:?`, `:`, `??`, `%`, `%?`, `^`, `^!`, `^>`, `<-` (loop), `!{`, `!!`, `!^`, `!&`, `=>` |
| Declaration sigil | `hl-decl` | `@`, `~@`, `*@`, `@/`, `@$`, `@?`, `@=`, `+@`, `$`, `$$`, `$=`, `+$` |
| Import / module | `hl-imp` | `<`, `<.`, `@target`, `@main` |
| Decorator | `hl-dec` | `#[...]` |
| Function / method name | `hl-fn` | `render_blocks`, `init`, `summary` |
| Class / type name | `hl-cls` | `Page`, `Task`, imported class names in type positions |
| Type shorthand / annotation | `hl-type` | `i`, `f`, `s`, `b`, `B`, `A`, `N`, `->`, compound types |
| Builtin / shorthand | `hl-bi` | `p`, `@enum`, `@zip`, `@map`, `@flt`, `@srt`, `@str`, `@int`, `@flt_n` |
| Self / super | `hl-self` | leading `.attr`, `..init` |
| Boolean / null constant | `hl-const` | `T`, `F`, `N`, `U` |
| Number | `hl-num` | `42`, `3.14`, `0xff` |
| String / raw / bytes | `hl-str` | `"..."`, `'...'`, `_"..."`, `0b"..."`, `` `...` `` |
| Interpolation | `hl-interp` | `{`, `}`, and `:.2f` inside `` `...` `` |
| Operator | `hl-op` | `\|>`, `:=`, `&&`, `\|\|`, `===`, `!==`, `..`, `:?`, `#` (len), `!` (not) |
| Punctuation | `hl-punc` | `{`, `}`, `(`, `)`, `[`, `]`, `,`, `;`, `:` |
| Comment | `hl-com` | `# comment` |

Plain identifiers and whitespace are emitted without a span. Adjacent tokens of the same
class are merged, so `<span class="hl-kw">?</span><span class="hl-kw">:</span>` never
appears for a single logical run.

### Type positions are contextual

Type shorthands are highlighted as `hl-type` only where they are types:

```myna
@fn(i:i, s:s)->b { ^T }   # parameter names i/s are plain; annotations are types
i = 0                    # plain local variable
s = "hello"              # plain local variable
ok = x :? [s]            # s is a type in a runtime check
$= Alias = [Task]        # Alias and Task are type-position names
```

## Themes

```bash
./bin/mynashine css                      # MynaPress light/dark
./bin/mynashine css --theme light        # fixed light
./bin/mynashine css --theme dark         # fixed dark
./bin/mynashine css --theme high-contrast
```

The `mynapress` theme defines `--hl-*` custom properties on `:root`, follows
`prefers-color-scheme`, and reuses MynaPress's `--code-bg`/`--fg` when present. All token
colours used by the built-in themes meet WCAG AA (4.5:1) against their code backgrounds.

Line numbers are opt-in:

```myna
html = highlight_code(code, "myna", T)
# <span class="hl-line" data-line="1">...</span>
```

The stylesheet renders `data-line` with a pseudo-element, so line numbers are not part of
the copied text.

## Architecture

```text
src/
  main.myna          # CLI entrypoint
  cli.myna           # argument parsing and dispatch
  token.myna         # [kind, text] tokens, span merging, lossless rendering
  myna_lexer.myna    # context-aware Myna scanner
  lang_py.myna       # Python scanner
  lang_js.myna       # JavaScript scanner
  lang_config.myna   # TOML / YAML / front matter
  lang_shell.myna    # Bash / shell / console
  lang_markup.myna   # HTML / XML / Markdown / MynaPress templates
  highlight.myna     # public API + HTML document post-processor
  theme.myna         # CSS variables and theme generation
  site_proc.myna     # build-directory walker + idempotent CSS injection
  demo.myna          # showcase generator
  fsutil.myna        # small filesystem helpers (Python target only)
  util.myna          # portable escape/unescape, character classification
tests/
  run_tests.myna
  harness.myna
  test_myna_lexer.myna
  test_companions.myna
  test_html_proc.myna
  test_lossless.myna
  test_site.myna
```

The core scanners, token layer, renderer, theme generator, and public highlighter declare
`@target(py, js)` and avoid Python-only APIs. The CLI/filesystem modules declare
`@target(py)`.

### Lossless invariant

For every input `code`:

```text
html_unescape(strip_tags(highlight_code(code, lang))) == code
```

This holds for valid code and malformed code, with and without line numbers. The test suite
checks it against every `.myna` file in this repository, `../myna/examples/`,
`../mynapress/src/`, and a corpus of adversarial fragments.

### Fault tolerance

Unrecognised characters become plain escaped text. Unterminated strings/comments/templates
run to EOF and are still emitted. No scanner raises on arbitrary input; the malformed-input
suite exercises every language against unterminated quotes, backticks, braces, comments,
stray sigils, and non-ASCII text.

## Development

```bash
# From the repository root:
python3 -m myna run tests/run_tests.myna

# Or against a local Myna checkout:
PYTHONPATH=/home/joseph/Projects/myna python3 -m myna run tests/run_tests.myna

# Generate the showcase:
python3 -m myna run src/main.myna demo -o build/demo.html
```

The test entrypoint lives in `tests/`, and Myna resolves relative imports from the entry
file's directory. `tests/src -> ../src` is a symlink that makes `<.src...` resolve to the
repository source tree. See `GAPS.md` G3.

`python3 -m myna stats src/main.myna --show` reports token statistics for a file (the
`stats` command takes one input file, not a directory).

## Status and non-goals

v0.1 targets **HTML + CSS highlighting for fenced code blocks**, primarily MynaPress. It is
not a VS Code extension, TextMate grammar, Tree-sitter grammar, LSP semantic-token
provider, or browser-side highlighter.

Known Myna toolchain friction is recorded in [`GAPS.md`](GAPS.md). The most notable item is
that Myna's JavaScript target currently rejects relative imports, so the pure core can be
compiled to JS only when bundled or rewritten with a single-file import strategy.

## License

MIT. See [`LICENSE`](LICENSE).
