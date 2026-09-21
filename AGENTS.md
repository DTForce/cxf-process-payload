# AGENTS.md

Guidance for AI coding agents working in this repository. Written to the
[agents.md](https://agents.md) convention, so any agent that reads `AGENTS.md` can use it.
Human contributors want `README.md` instead.

## What this is

A VS Code extension (`dtforce.cxf-payload-process`) that turns an Apache CXF request/response
log line — a one-line header block ending in `Payload: <...>` — into something readable. The
payload is unescaped, un-CDATA'd and pretty-printed, and the document is switched to the `cxf`
language so the embedded XML/JSON gets highlighted.

## Commands

```bash
npm install          # node_modules is gitignored and not checked out by default
npm run compile      # tsc -p ./   → out/extension.js (the manifest's main entry point)
npm run watch        # incremental tsc; also the preLaunchTask for debugging
npm run lint         # tslint ./src/*.ts — currently passes; keep it that way
npx @vscode/vsce ls  # list exactly what would go into the .vsix
```

There are no tests and no test runner. To exercise a change by hand, press `F5` (launch config
**Extension**) for an Extension Development Host. It runs `npm: watch` first, so `out/` has to
compile before the host picks anything up.

Bump `version` in `package.json` as its own commit, as the history does.

## Architecture

Everything lives in `src/extension.ts`. `processXmlText()` is the single transformation, and
all three entry points funnel into it:

1. **Command** `processCXF.processPayloadXML` ("Process Payload (XML)") — rewrites the current
   selection in place, or the whole document when the selection is empty, then calls
   `setTextDocumentLanguage(..., 'cxf')` and scrolls to the top.
2. **Formatter** — a `DocumentFormattingEditProvider` registered for the `cxf` language, so
   `Format Document` on a `.cxf`/`.req`/`.resp` file re-runs the same processing.
3. **URI handler** — `vscode://dtforce.cxf-payload-process/base64/<base64>` decodes the payload
   and opens it in a new untitled document, so an external tool can hand a payload to the editor.

### The `processXmlText` pipeline

Input is a CXF log line whose header fields (`ID:`, `Address:`, `Content-Type:`, `Payload:` …)
are space-separated on one line. In order:

1. Insert a newline before each `Key: ` field that precedes `Payload: `, splitting the flat
   line into a header block.
2. Split at the first `Payload:`; everything after it is the payload. No `Payload:` means the
   whole input is treated as the payload.
3. XML branch (payload starts with `<`): HTML-decode entities, strip an `<?xml?>` declaration
   nested inside CDATA, unwrap `<![CDATA[` / `]]>`, then `tsxml`'s `Compiler.formatXmlString`.
   **If that throws it retries on the raw, un-preprocessed payload** — the preprocessing is
   lossy on unusual documents, so the fallback is deliberate, not dead code.
4. Otherwise: `JSON.stringify(JSON.parse(payload), null, 4)`.
5. A post-format pass re-inlines elements whose text content is short (≤30 chars) and contains
   no markup, so trivial leaf elements do not each eat three lines.

The regexes in steps 1, 3 and 5 are the fragile part of this extension and the subject of most
of the fix commits. They use lookbehind and the `s` flag — ES2018 features TypeScript does not
downlevel, so they work only because the extension host runs a modern Node. The `es6` target in
`tsconfig.json` says nothing about them, and raising it is a prerequisite for any TypeScript 5
upgrade, which rejects the `s` flag under an ES6 target.

Note that `.replace("<![CDATA[", "")` and `.replace("]]>", "")` take string arguments, so only
the first occurrence of each is removed. That is fine for a single-payload document and a known
limit if a payload nests several CDATA sections.

### Known defects

Documented in the README under "Known limitations" and reproduced by the cases below. Do not
"fix" the surrounding code without checking these first, because several look like bugs but are
the documented behaviour:

- `tsxml` strips namespace prefixes: `<soap:Envelope>` formats back as `<Envelope>`. The output
  is therefore not equivalent to the input. This is the strongest argument for replacing `tsxml`.
- The header-splitting regex is `[A-Za-z]+`, so hyphenated names (`Content-Type:`,
  `Http-Method:`) never get their own line.
- A fully escaped payload (`&lt;root&gt;…`) does not start with `<`, so it takes the JSON branch
  and throws.
- With no `Payload:` field, `offset` is `-1`, so `input.substring(0, -1)` returns `''`: the text
  is dropped and a bare `Payload: ` prefix is injected.
- The URI handler calls `atob`, a Node 16 global, so it needs VS Code ≥ 1.67 despite the
  manifest's `^1.32.0`. It type-checks only because `target: es6` pulls in the DOM lib.

### Verifying a change to the formatter

There is no test runner, so check behaviour by extracting `processXmlText` (everything in
`src/extension.ts` above `activate()`, minus the `vscode` import), compiling it standalone and
running representative payloads through it: a SOAP envelope with namespace prefixes, a CDATA
body containing its own XML declaration, an escaped body, a JSON body, and input with no
`Payload:` field. Diff the output before and after the change. Those five cover every branch.

### Language contribution

- `syntaxes/cxf.grammar.json` — TextMate grammar for `source.cxf`. It matches the `Key: value`
  header lines and, after `Payload:`, delegates to the embedded `text.xml` or `source.json`
  grammar depending on whether the payload's first interesting character is `<` or `{`. Both
  embeddings are also declared in `package.json` under `grammars.embeddedLanguages` — a change
  in one must be mirrored in the other.
- `language-config.json` — brackets and auto-closing pairs for `cxf`; comment syntax is
  borrowed from INI.
- Extensions bound to `cxf`: `.cxf`, `.req`, `.resp`.

## Dependency pins that matter

`@types/vscode` is `~1.32.0`, not `^1.32.0`, and must stay that way while TypeScript is on 3.x.
A caret range lets it float to the newest 1.x, whose `.d.ts` uses syntax TypeScript 3.9 cannot
parse; the build then fails with a wall of TS1005 errors inside
`node_modules/@types/vscode/index.d.ts`. It also has to stay `<=` `engines.vscode`, or
`vsce package` refuses to build.

`package-lock.json` had gone five years without regeneration, which is where every Dependabot
alert on this repo came from — the declared ranges already permitted the patched versions. If
alerts reappear, check whether a plain `rm package-lock.json && npm install` clears them before
touching any declared version.

`tsxml` is unmaintained (one version ever published, 0.1.0, March 2017) and `tslint` has been
deprecated since 2019. Replacement paths for both are in the README under "Dependencies".

## Packaging

`.vscodeignore` keeps sources, build configuration and contributor docs out of the `.vsix`, and
additionally drops `tsxml`'s bundled test suite and TypeScript sources, which its package `main`
never loads. Its patterns are rooted rather than written with `**` on purpose: a glob such as
`**/*.ts` would reach into `node_modules` and strip files some future dependency might need at
runtime. Keep new patterns rooted, and confirm with `npx @vscode/vsce ls` that
`node_modules/tsxml/build/` is still fully present after editing it.

## Conventions

- Source is indented with **tabs** (`.vscode/settings.json` sets `editor.insertSpaces: false`),
  enforced by tslint along with mandatory semicolons. `npm run lint` passes — do not regress it.
- `strict` TypeScript is on.
- Licensed Apache-2.0, matching the other public DTForce repositories. `package.json` carries
  the matching SPDX `license` field. Every dependency, development ones included, is MIT, ISC,
  BSD-2-Clause, BSD-3-Clause, 0BSD or Apache-2.0, so there is no copyleft obligation; re-check
  this before adding a dependency. The bundled MIT dependencies keep their own terms and their
  licence files must stay in the `.vsix` — do not exclude them in `.vscodeignore`.
