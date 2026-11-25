Vendored from [@duckduckgo/autoconsent](https://github.com/duckduckgo/autoconsent)
16.40.0, under MPL-2.0 (`autoconsent-LICENSE`). Unmodified.

- `autoconsent.js` — `dist/autoconsent.playwright.js`, the build meant to be
  injected into an isolated world. It calls `window.autoconsentSendMessage` and
  expects replies through `window.autoconsentReceiveMessage`; `arb consent`
  implements that bridge.
- `autoconsent-rules.json` — `rules/compact-rules.json`, the consent-tool rules.

To update, refetch both files from
https://cdn.jsdelivr.net/npm/@duckduckgo/autoconsent@<version>/ and bump the
version above.
