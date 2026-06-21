# SAP Help Portal Fetch Protocol (browserless)

**For retrieving AUTHORITATIVE official SAP documentation text from help.sap.com.**

`help.sap.com` is a JavaScript SPA — plain `WebFetch`/`curl` on a doc URL returns an empty shell or "Page Not Found", NOT the content. Two bundled Node scripts retrieve the real text without a browser. Use them whenever you would otherwise cite help.sap.com from memory.

> Node only, no extra deps, no auth. Run with the plugin's bundled Node.

## Which script to use

| You need… | Script | Input |
|---|---|---|
| **ABAP language / keyword reference** (SELECT, syntax, statements, ABAP types) | `scripts/fetch-abap-keyword-doc.mjs` | topic id (`abenwhere_all_entries`) or any abapdocu URL |
| **Functional / module / config / process docs** (SD pricing, FI dunning, MM release strategy, IMG concepts, Fiori app help) | `scripts/fetch-sap-help-doc.mjs` | a full `help.sap.com/docs/<product>/<deliverable>/<topic>.html` URL |

```bash
node "$CLAUDE_PLUGIN_ROOT/scripts/fetch-abap-keyword-doc.mjs" abenwhere_all_entries
node "$CLAUDE_PLUGIN_ROOT/scripts/fetch-sap-help-doc.mjs" "https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/<deliverable>/<topic>.html"
```
Both print the official body text (description, restrictions, examples / config steps) plus the source URL for citation.

## How to find the URL (when you only have a topic)

1. `WebSearch` the topic restricted to help.sap.com, e.g. `MM release strategy purchase order help.sap.com` or `SELECT FOR ALL ENTRIES help.sap.com`.
2. Take the resulting `help.sap.com` URL:
   - `…/doc/abapdocu_…/<topic>.htm` → pass the `<topic>` (or the URL) to **fetch-abap-keyword-doc.mjs**
   - `…/docs/<product>/<deliverable>/<topic>.html` → pass the full URL to **fetch-sap-help-doc.mjs**
3. Run the matching script; cite the printed Source URL.

## Why these work (so you can fix/extend)

- **ABAP keyword docs**: content is embedded in the `.html` as a `new sap.ui.model.json.JSONModel({ par1, … })` literal (note `.html`, not the `.htm` SPA route). The script extracts it.
- **Functional docs**: the `.html` is an empty shell; content arrives via the `http.svc` JSON API. The script chains `deliverableMetadata` (→ `data.deliverable.id`) then `pagecontent` (→ `data.body`).

## Rules

- **Cite the Source URL** the script prints. Never present help.sap.com content from memory when the script can fetch it.
- **Specify the SAP release.** The functional fetcher prints the resolved version; if it resolved from `LATEST` that is a FALLBACK, not authoritative — for release-specific guidance pass `?version=<rel>` in the URL and confirm it matches the project's `.sc4sap/config.json` release (ECC vs S/4HANA).
- **Role split.** Module consultants use `fetch-sap-help-doc.mjs` for their OWN module's functional/config docs only; ABAP keyword/language lookups and deep cross-topic doc research belong to `sap-doc-specialist`.
- **Script path.** Prefer `"$CLAUDE_PLUGIN_ROOT/scripts/<script>"`. If `$CLAUDE_PLUGIN_ROOT` is unset, the installed copy lives at `~/.claude/plugins/cache/sc4sap/sc4sap/<version>/scripts/<script>` (glob the newest `<version>` dir). Last resort — the manual fallback: for ABAP, `curl` the `.html` and read the `par*/ul*/code*` strings; for functional, call `http.svc/deliverableMetadata` (→ `data.deliverable.id`) then `http.svc/pagecontent` (→ `data.body`).

## Scope / limits (be honest)

- ✅ help.sap.com ABAP keyword docs + application/functional/config docs.
- ❌ **OSS Notes (me.sap.com)** — auth-walled (S-user login). NOT retrievable by these scripts; WebSearch for the note + state plainly that full text needs SAP support login.
- ❌ The `/docs/r/…` readable-URL variant — pass the canonical `/docs/<product>/<deliverable>/<topic>.html` form instead.
