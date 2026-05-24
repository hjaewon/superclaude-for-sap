# Program → Spec — Workflow Steps

Referenced by `SKILL.md`. Follow these 6 steps (Step 0 through Step 5) whenever the skill runs.

**Step 0 — Socratic interview** (see `Socratic_Scope_Narrowing` section in `SKILL.md`)
Default opener: issue ONE bundled `AskUserQuestion` call with the four standard questions — **Audience / Format / Depth / Language** — in that exact order, each single-select with the "(Recommended)" option first. This is MANDATORY whenever the target object is already present in `ARGUMENTS`; it replaces Rounds 2+3+5 in a single UI turn.
Fall back to per-round questioning only when (a) the object is missing/ambiguous (run Round 1 first) or (b) the user picks L3/L4 in the bundle (run Round 4 scope-trimming after).
Never skip entirely unless the user supplies `object=... depth=L2 format=md lang=ko` style fully-qualified arguments.

**Step 1 — Inventory** (auto, parallel MCP calls)
- `SearchObject` — confirm object + sub-type
- Metadata: `GetObjectInfo` — package, author, created/changed, transport

**Step 1.5 — CBO inventory lookup** (auto)
- Resolve `<PACKAGE>` from `GetObjectInfo` above.
- Ask the user one question: "Which module does package `<PACKAGE>` belong to? (SD / MM / PP / PM / QM / WM / TM / TR / FI / CO / HCM / BW / PS / Ariba)" — only if the module cannot be derived from `.sc4sap/config.json` or the package's existing CBO folder.
- Check `.sc4sap/cbo/<MODULE>/<PACKAGE>/inventory.json`.
  - **Exists** → Load it. When the analyst describes data sources, tables, or helper calls in Step 3, annotate each one that matches an inventory entry with its CBO role + one-line business purpose (e.g., "writes to `ZSD_ORDER_LOG` — append-only sales-order processing log"). This turns opaque Z-references in the spec into named reusable assets.
  - **Missing** → Print one line: "No CBO inventory at `.sc4sap/cbo/<MODULE>/<PACKAGE>/`. Run `/sc4sap:analyze-cbo-obj` first for richer spec annotations, or type `skip` to proceed."
- Persist the loaded entries to `.sc4sap/specs/<OBJECT>/cbo-context.md` so sap-analyst and sap-writer consume it in Step 3.
- Source:
  - Report/Program: `GetProgFullCode` + `GetIncludesList` → iterate `GetInclude`
  - Class: `ReadClass` (all sections) + `GetLocalDefinitions` / `GetLocalMacros` / `GetLocalTestClass` / `GetLocalTypes`
  - Function Module: `ReadFunctionModule` + function group includes
  - CDS: `ReadView` + `GetMetadataExtension`
  - RAP: `Read BehaviorDefinition` + `Read BehaviorImplementation` + `Read ServiceDefinition` + `Read ServiceBinding`
- Screens / GUI Status / Text Elements (if report / module pool): `GetScreensList`, `GetGuiStatusList`, `GetTextElement`
- Structural: `GetAbapAST`, `GetAbapSemanticAnalysis`
- Enhancements (L3+): `GetEnhancements`, `GetEnhancementSpot`
- Where-Used (L4 only — fixed scope): `GetWhereUsed` against the main object **plus each screen**; filter callers to customer namespace `Z*` / `Y*` only. Skip standard SAP and add-on namespaces.

**Step 2 — Classify** (auto)
- Object archetype: ALV report / batch job / BDC / FM wrapper / CDS view / RAP BO / enhancement impl / utility class
- Drives which spec template is applied in Step 3.

**Step 3 — Delegate to sap-analyst + sap-writer** (+ sap-critic on L4)

Emit Phase Banner before each dispatch (see `SKILL.md` § Phase_Banner):

```
▶ phase=3.analyst · agent=sap-analyst · model=Opus 4.7
▶ phase=3.writer · agent=sap-writer · model=<Haiku 4.5 for L1/L2 | Sonnet 4.6 override for L3/L4>
▶ phase=3.critic (L4 only) · agent=sap-critic · model=Opus 4.7
```

- **sap-analyst** (Opus 4.7, frontmatter) extracts: business purpose, inputs (selection screen / importing params), outputs (ALV cols / exporting params / OData entity), data sources (tables + CDS + BAPIs), main logic narrative, error cases, authorization checks (`AUTHORITY-CHECK` statements). When `cbo-context.md` exists, the analyst cross-references every Z-object mentioned against the inventory and replaces opaque "Z-table" / "Z-class" labels with the inventory's documented role + business purpose.
- **sap-writer** (Haiku 4.5 base; **`model: "sonnet"` override for L3/L4 depth** — longer narrative + deeper cross-reference + stronger consistency requirement) renders into the chosen format (MD or Excel) at the chosen depth + language.
- **sap-critic** (Opus 4.7, frontmatter) gate (only if L4): verifies every claim cross-references a line range.

**Step 3.5 — Draw screens (Markdown only)**

Excel output inherits the two reference mockup PNGs already embedded in `asset/template_base.xlsx` (Selection screen at C4, ALV grid at C19 on the `입력 및 화면` sheet) — drivers must NOT regenerate them. Custom per-program PNG replacement is a separate user-triggered task (see [`spec-templates.md`](spec-templates.md) § Image Replacement).

For **Markdown output only**, render every Selection-Screen and output Screen / ALV as an ASCII wireframe inside fenced code blocks (character widths are uniform there):
- Reconstruct Dynpros from `GetScreen` / `ReadScreen` (`HEADER` + `FLOW_LOGIC` + field positions `LINE` / `COLUMN` / `LENGTH` / `HEIGHT`).
- Reconstruct Selection-Screens from `PARAMETERS` / `SELECT-OPTIONS` / `SELECTION-SCREEN BLOCK` statements.
- Reconstruct ALV output from the field catalog (columns + widths + headings).
- Show label + input box + F4-help marker `[▼]` + mandatory `*`.
- Optional Mermaid `flowchart TD` for screen-to-screen navigation.
- For GUI Status: short FKEY → FCODE → text table plus an ASCII toolbar bar.

For objects without UI (pure class, FM, CDS, RAP without screens), skip the wireframes — the Parameters table inside the Inputs section is enough.

**Step 4 — Render**
- **Markdown**: single `.md` with H2 sections per spec dimension, tables for selection-screen / tables / methods / exits. See [`spec-templates.md`](spec-templates.md) for the section skeleton.

- **Excel (MANDATORY workflow — 양식 보존 + program-specific imagery, single entry point)**:

  > **Why clone + image swap?** Geometry (styles / borders / fonts / column widths / row heights / drawings) comes from `asset/template_base.xlsx` clone — that's what prevents the old throwaway-driver drift. Per-program data flows in through TWO inputs: (1) a TR (translation) map that replaces the template's English strings, and (2) an image-spec that drives the per-program Selection / ALV / Process-Flow mockups. Both run on every Excel spec — no opt-in trigger keywords. sap-writer's job is to produce both JSON files; one helper does the rest.

  Pipeline for each Excel-output spec:

  1. **sap-writer produces TWO JSON files**:
     - `.sc4sap/specs/_tr/{OBJECT}-{YYYYMMDD}.tr.json` — flat `{ "English key": "한국어 값" }` map. Schema + slot semantics in [`spec-templates.md`](spec-templates.md) § Excel — Template-clone.
     - `.sc4sap/specs/_img/{OBJECT}-{YYYYMMDD}.image-spec.json` — `renderScreenImages()` argument: `{ selection: {fields:[…]}, alv: {columns:[…], sampleRows:[…]}, processFlow: [string,…], lang }`. Exact key names in [`spec-templates.md`](spec-templates.md) § Image Replacement § Programmatic. Schema mistakes (e.g. `field` instead of `name`, array sampleRows instead of objects) silently render empty PNGs — verify by inspecting the resulting ALV byte size (~12 KB normal, ~1 KB = empty grid).
  2. **Run the single entry point**:
     ```bash
     node scripts/spec/build-spec.mjs <tr.json> <image-spec.json> <out.xlsx>
     ```
     Internally: `cloneTemplate(tr)` → `renderScreenImages(imageSpec)` → `swapImages(xlsxPath, …pngBuffers)`. Default output path is `.sc4sap/specs/{OBJECT}-{YYYYMMDD}-{lang}.xlsx`. Pass `-` for the image-spec argument to skip image rendering and ship the text-only spec with the template's generic mockups (rare — only when no per-program imagery makes sense).
  3. **Verify the artifact** — output size ≈ 95–110 KB depending on PNG sizes. `unzip -l` lists `xl/sharedStrings.xml` + `xl/media/image1.png` + `image2.png` + `image3.png` + `xl/drawings/drawing3.xml` + `drawing4.xml`. Open in Excel and scan every sheet — geometry MUST match `asset/template_base.xlsx`, Sheet 3 shows the program-specific Selection + ALV, Sheet 4 shows the horizontal Process Flow under the heading.
  4. **Cleanup** — leave both JSON files in `_tr/` and `_img/` for traceability. Remove only ephemeral files (HUD probes, smoke tests).

  **Graceful degrade** — when no headless browser is on PATH (Chrome / Edge / Chromium not installed), `renderScreenImages` returns `null` per slot and `swapImages` skips them. The xlsx ends with template generic mockups on Sheet 3 and a blank Sheet 4 drawing — never crashes.

  **Zero external npm dependencies** — `build-spec.mjs` / `template-clone.mjs` / `image-swap.mjs` / `xlsx-zip.mjs` use only `node:fs` / `node:zlib` / `node:path` / `node:url`. Image rendering uses `screen-image-renderer.mjs` which shells out to a system headless browser; no npm modules.

  **Sheet order is fixed by `asset/template_base.xlsx`** — clone never reorders. The template ships with:
  1. `프로그램 개요` / Program Overview — Field/Value metadata (17 rows)
  2. `데이터 모델` / Data Model — Table/Access/Key Fields/Join Type/Notes (4 table slots + trailer)
  3. `입력 및 화면` / Inputs & Screens — Parameters (5 slots) + 5 warning rows + image anchors at C4 (Selection) / C19 (ALV)
  4. `처리 로직` / Processing Logic — #/Event/Step (12 step slots) + Process Flow Chart heading at B18 + horizontal flow image at B19
  5. `출력` / Output — Order/Field/Description/Length/Edit/Hidden (10 column slots)
  6. `권한` / Authorizations — Check/Object/Level/Implemented?/Notes (5 rows)
  7. `예외 처리` / Exceptions — Trigger/Mechanism/Message/Recovery (3 rows)

  **Image anchor extents are dynamic** — each `<xdr:ext>` is computed from the supplied PNG's IHDR (`px × 9525` EMU) so PNGs render at native aspect ratio without stretching. Sheet 3 anchors (drawing3.xml C4 + C19) are surgically updated by image name. Sheet 4 (drawing4.xml) is injected on demand because each program's flow chart differs (`xl/media/image3.png` + `<xdr:oneCellAnchor>` from B19 + `_rels/drawing4.xml.rels`).

**Step 5 — Review loop**
- Show a table of contents + first section inline.
- Ask: "OK to finalize, or trim/expand a section?"
- On confirm → write file → print absolute path.
