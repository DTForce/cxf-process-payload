# CXF Process Payload

A small Visual Studio Code extension that makes Apache CXF request/response logs readable.

CXF writes each interaction as a single, very long line — every header crammed together and
the whole SOAP or JSON body escaped inline:

```
ID: 4 Address: http://example.com/ws/OrderService Http-Method: POST Content-Type: text/xml Headers: {} Payload: <soap:Envelope xmlns:soap="..."><soap:Body><ns2:getOrder ...><orderId>12345</orderId>...
```

Paste that into an editor, run **Process Payload (XML)**, and you get this instead:

```
ID: 4
    Address: http://example.com/ws/OrderService Http-Method: POST Content-Type: text/xml
    Headers: {}
    Payload: <Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <Body>
    <getOrder xmlns:ns2="http://example.com/orders">
      <orderId>12345</orderId>
      <customer>
        <name>Jan Novak</name>
        <vip>true</vip>
      </customer>
    </getOrder>
  </Body>
</Envelope>
```

The document is also switched to the `cxf` language, so the headers and the embedded
XML/JSON get syntax highlighting.

## Install

Not published to the Marketplace. Build a `.vsix` and install it locally:

```bash
npm install
npx @vscode/vsce package
code --install-extension cxf-payload-process-0.3.2.vsix
```

## Usage

There are three ways in, and all of them run the same transformation.

**1. The command.** Open the Command Palette and run **Process Payload (XML)**
(`processCXF.processPayloadXML`). It rewrites the current selection, or the whole document
if nothing is selected, then scrolls back to the top.

**2. Format on a `.cxf` file.** Files ending in `.cxf`, `.req` or `.resp` are recognised as
the `cxf` language, and **Format Document** re-runs the same processing. Handy if you dump
logs straight to disk.

**3. A URI, for handing a payload over from another tool.** Opening

```
vscode://dtforce.cxf-payload-process/base64/<base64-encoded-log-line>
```

decodes the payload and opens it, formatted, in a new untitled document. This is the hook
a log viewer or a shell script would use:

```bash
open "vscode://dtforce.cxf-payload-process/base64/$(base64 < payload.txt | tr -d '\n')"
```

## What it actually does

1. Splits the flat log line into a header block, putting each `Key: value` field on its
   own line.
2. Takes everything after the first `Payload:` as the body. If there is no `Payload:`
   field, the whole input is treated as the body.
3. If the body starts with `<`, it is XML: HTML entities are decoded, a `<![CDATA[ ]]>`
   wrapper and any XML declaration nested inside it are stripped, and the result is
   pretty-printed.
4. Otherwise it is parsed as JSON and re-printed with 4-space indentation.
5. Finally, elements whose text content is short (30 characters or less) and contains no
   markup are pulled back onto one line, so trivial leaf elements don't each eat three.

## Known limitations

These are real and worth knowing before you trust the output:

- **Namespace prefixes are stripped.** `<soap:Envelope>` comes back as `<Envelope>` and
  `<ns2:getOrder>` as `<getOrder>` — the `xmlns:` declarations survive, but the prefixes on
  the elements do not. The formatted document is therefore *not* equivalent to the input.
  Fine for reading; do not copy the result back into a request. This comes from the
  underlying `tsxml` formatter.
- **Hyphenated headers are not split out.** `Content-Type:`, `Http-Method:` and
  `Response-Code:` stay glued to the preceding field, because the splitting pattern only
  recognises alphabetic header names.
- **A fully escaped payload fails.** If the body arrives as `&lt;root&gt;...` rather than
  `<root>...`, it does not start with `<`, so it is sent down the JSON path and throws.
- **Input with no `Payload:` field** gets a spurious `Payload: ` prefix added and the text
  before it dropped.
- **The `base64` URI handler needs VS Code 1.67 or newer**, even though the manifest claims
  1.32. It calls `atob`, which only became available in the extension host when VS Code
  moved to Node 16.

## Development

```bash
npm install
npm run compile      # tsc -p ./  →  out/extension.js
npm run watch        # incremental build; also the F5 pre-launch task
npm run lint         # tslint; passes as of this commit
```

Press <kbd>F5</kbd> to open an Extension Development Host with the extension loaded. There
are no automated tests.

`.vscodeignore` controls what ends up in the `.vsix`; run `npx @vscode/vsce ls` to see the
exact file list before publishing.

All the logic lives in `src/extension.ts`, in a single `processXmlText()` function that the
command, the formatter and the URI handler all call. The grammar is in
`syntaxes/cxf.grammar.json`; it delegates to the built-in XML and JSON grammars for the
payload, depending on the first character it sees.

Be careful with the regular expressions in `processXmlText()` — they use lookbehind and the
`s` flag, and they are the source of most of the behaviour above. The XML formatter is
wrapped in a `try`/`catch` that retries on the raw, un-preprocessed payload; that fallback
is deliberate, because the preprocessing is lossy on unusual documents.

## Dependencies

Runtime dependencies are deliberately minimal — `html-entities` for entity decoding and
`tsxml` for XML formatting. Neither is bundled with a bundler; `vsce` ships them as-is, along
with their MIT licence files.

Two things a future maintainer should plan for:

| Package | Status | Suggested path |
|---|---|---|
| `tsxml` | Unmaintained. One version ever published (0.1.0, March 2017) and it is what strips namespace prefixes. | Replace with `xml-formatter` or `fast-xml-parser`. This is the fix for the biggest correctness problem, but it will change output, so re-check the inlining step in step 5. |
| `tslint` | Deprecated since 2019 in favour of ESLint. | Move to `eslint` + `typescript-eslint`. Verified to work on this codebase; it reports 3 trivial issues, all auto-fixable. |

Upgrading TypeScript past 3.x additionally requires raising `target` in `tsconfig.json`
from `es6` to `es2018`, because TypeScript 5 rejects the `s` regex flag under an ES6 target.
`@types/vscode` is pinned with `~` on purpose: the caret range it used to carry allowed it
to float to a version whose type definitions the pinned TypeScript could no longer parse.

## License

Apache License 2.0 — see [LICENSE](LICENSE).

The bundled runtime dependencies keep their own terms: `html-entities` and `tsxml` are both
MIT, and their licence files are shipped inside the `.vsix`. The full dependency tree,
development dependencies included, is MIT, ISC, BSD-2-Clause, BSD-3-Clause, 0BSD or
Apache-2.0 — all permissive, with no copyleft anywhere.
