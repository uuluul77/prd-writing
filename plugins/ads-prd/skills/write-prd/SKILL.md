---
name: write-prd
description: >
  Write a first draft of an Ads API PRD for achieving feature parity with TikTok platform features (TikTok One / TTO, TikTok Ads Manager). Use this skill when Winnie wants to draft or fill in a PRD, has source materials (platform PRD, tech design doc, or API docs), and needs the PRD sections written. Triggers: "write PRD", "draft PRD", "fill in the PRD", "write up the PRD", "write a PRD for", "PRD draft".
---

# Write PRD — Ads API Feature Parity

## ⛔ Hard Stops — read before anything else

These are absolute prohibitions. Active at every step. No exception, no "it seemed obvious."

**#1 — § 1 Meego Configuration: never touch it.**
When assembling the draft, copy the existing § 1 block verbatim from the Lark doc. Do NOT fill in any blank cells. Do NOT change any existing values. If it was blank when you opened the doc, it stays blank.
*One exception, and only with Winnie's explicit say-so:* **Project Priority**. See § Priority alignment rule — a silent mismatch between § 1 and Basic Info has shipped to Meego before.

**#2 — New fields / endpoints: never name them yourself.**
Any field or endpoint path not already in the public API docs must use `⚠️⚠️⚠️ [field name TBD — to be confirmed by Winnie] ⚠️⚠️⚠️` as a placeholder. Ask via AskUserQuestion before writing the name. Then ask separately: scalar or object? If object, get all sub-field names, types, and constraints before writing any rows. See § New field / endpoint naming rule.

**#3 — Endpoint lines: always the full URL.**
Write `POST https://business-api.tiktok.com/open_api/v1.3/ad/create/`.
Never just the path: `POST /open_api/v1.3/ad/create/`.

**#4 — Questions: always AskUserQuestion.**
Never write a plain-text question ending in `?` to the user. See § Interaction style.

**#5 — Pre-push checklist: run it every time.**
Before every push, verify all 5 items in § Push to Lark. Hard stop on item 1 (Meego).

**#6 — Field names come from the public API doc, not the tech design.**
Tech design docs use internal/backend field names that are never exposed to developers. Derive parameter names and descriptions from the public `business-api.tiktok.com` page (or, for alpha endpoints, the external Lark spec doc). If no doc exists yet, use `⚠️⚠️⚠️ [TBD] ⚠️⚠️⚠️` — never carry over internal names.

**#7 — Parameter and response tables use L1–L4 nesting columns.**
This is Winnie's standard shape and she has asked for it in *every* PRD to date. Build it that way the first time. See § Table format.

**#8 — Field descriptions are verbatim, never summarized.**
Copy the source description in full — including its Note, Example, Data source, Updated in, and enum lines — and keep the source's own extra columns (e.g. `Required permission`). Condensing gets rejected. See § Table format.

---

## Interaction style

**Every clarifying question goes through `AskUserQuestion`.** No plain-text questions to the user, ever.

Two hard rules:
1. **Free-text inputs use AskUserQuestion + "Other".** Always include an explicit option labeled `"I'll paste below"` (or similar) so the user knows the "Other" text field is where they type URLs / descriptions / schemas.
2. **If you find yourself about to write a plain-text question followed by a `?`, stop and reformulate as AskUserQuestion.**

Batch related questions into a single AskUserQuestion call (max 4 per call). Winnie often answers a multiple-choice question with free text that redefines the option — read the answer, don't pattern-match to the label you offered.

See § AskUserQuestion option templates for the reusable option sets.

---

## Step 0 — Pre-flight checks (run BEFORE asking the user anything)

Two cheap validations, in parallel, before any user-facing question.

### 0a. Auth health

```bash
lark-cli auth status | head -20
meegle auth status --format json     # only needed if this session will reach Step 6 / prd2meego
```

- `lark-cli` → needs `identities.user.available: true`. `tokenStatus: needs_refresh` is fine — it auto-refreshes on the next user API call. Confirm with a real fetch of any known-accessible doc.
- ❌ `"error": "invalid_client"` / `"code": 20069` → auth expired. Surface it and use the **"Auth expired"** template.

### 0b. Portal note (print as a line, not a question)

> 📌 `business-api.tiktok.com` doc pages are JS-rendered and **publicly accessible — no login**. I'll read them with the in-app browser via `document.body.innerText`.

> ⚠️ Never ask Winnie to log in to the portal, and never use the "Login-walled page" template for these URLs.

---

## Fetching docs — the fetch ladder

Use this **for every URL**. Do not pick a method ad-hoc.

### Tier 1: `WebFetch`
Fast, no browser. **Failure signal:** "consists only of a title", "no actual API documentation", or under ~500 chars of substance → JS-rendered SPA, escalate.

### Tier 2: in-app browser (`mcp__Claude_Browser__*`)

```
preview_start {url} → javascript_tool: document.body.innerText.length
```

**`business-api.tiktok.com` portal pages have NO iframe.** (Verified 08.2026 — this changed; older guidance told you to reach into `document.querySelector('iframe').contentDocument`, which now returns nothing and looks like a page-load failure.) `document.querySelectorAll('iframe').length === 0` and content renders directly in the main document — a typical endpoint page is 25K–110K chars of `innerText`.

**Two mechanics that will bite you:**

1. **Wrap every snippet in an IIFE.** The JS context persists between calls, so a bare `const s = ...` fails the second time with `Identifier 's' has already been declared`.
   ```js
   (function(){ const s = document.body.innerText; return s.substring(0, 2000); })()
   ```

2. **The `?=&` sanitize trick costs you verbatim text.** A slice containing `?`, `=`, or `&` can return `[BLOCKED: Cookie/query string data]`. Chaining `.replace(/[?=&]/g,'_')` dodges it — but it also mangles every example URL in the doc, and Hard Stop #8 needs those intact. So:
   - **Survey and locate with the sanitized read** (safe, never blocks).
   - **Re-fetch the few fields that contain URLs or query strings unsanitized, in small targeted slices** — short slices anchored on a keyword usually pass the filter cleanly.
   ```js
   (function(){ const s=document.body.innerText; const i=s.indexOf('Temporary URL for profile photo'); return s.substring(i, i+420); })()
   ```
   Note that sanitizing shifts nothing (same length) but *does* change content — don't mix offsets from a sanitized read with text from an unsanitized one without re-checking.

**Finding a field:** search first, then read around it.
```js
(function(){ const s=document.body.innerText.replace(/[?=&]/g,'_'); const i=s.indexOf('field_name_here'); return s.substring(i-200, i+900); })()
```

**Enumerate every occurrence** before assuming the first hit is the response-table one — a field name typically appears in the nav, the v1.2/v1.3 comparison table, the `fields` parameter list, the response body, *and* the permission changelog:
```js
(function(){ const s=document.body.innerText.replace(/[?=&]/g,'_'); const idx=[]; let p=-1; while((p=s.indexOf('is_business_account',p+1))!==-1) idx.push(p); return JSON.stringify(idx); })()
```

⚠️ JS state resets on navigation — capture data before navigating (return it from the tool, or write it to a file).

### Tier 3: ask the user to paste
Last resort. AskUserQuestion with `"I'll paste the schema below"`.

### Lark-specific notes

- **No proxy.** `lark-cli` v2 reaches CN (`bytedance.larkoffice.com`), US (`.us.`), MY (`.my.`), and SG (`.sg.`) hosts directly. Ignore any `ALL_PROXY=socks5://127.0.0.1:1080` instruction — that proxy is not running and is not needed.
- Fetch: `lark-cli docs +fetch --doc <url-or-token>`. Large docs: pipe through Python and slice, rather than Read's line chunking (single-line XML defeats offset/limit).
- **Always fetch comments too** — they carry Winnie's open questions and stakeholder asks:
  ```bash
  lark-cli api GET /open-apis/drive/v1/files/<token>/comments \
    --params '{"file_type":"docx","page_size":50}' --page-all
  ```
  For a **wiki** URL, `file_type: "wiki"` is invalid — resolve the node first via `GET /open-apis/wiki/v2/spaces/get_node --params '{"token":"<wiki_token>"}'`, then query with the returned `obj_token` and `file_type: "docx"`.
- Raw API syntax is positional: `lark-cli api GET <path> --params '<json>'`. There is no `--method` flag.

---

## AskUserQuestion option templates

| Situation | header | options |
|---|---|---|
| **Free-text input** | varies | `"I'll paste below"` + `"Skip — extract from docs"` |
| **Sign-off** | "Approve" | `"Yes, proceed"` + `"Adjust scope"` + `"Missing items — let me add"` + `"I'll describe changes below"` |
| **Fetch failed** | "Fetch" | `"Retry after I fix it"` + `"Paste content directly"` + `"Skip this source"` |
| **Auth expired** | "Lark auth" | `"I fixed it — retry"` + `"Skip Lark — paste-only mode"` |
| **Source selection (flow)** | "Flow source" | `"Use tech design doc"` + `"Use platform PRD"` + `"Same as existing flow"` + `"Not needed"` + `"I'll describe below"` |
| **Section review** | "Section" | `"Looks good — next section"` + `"Revise this section"` + `"Add more detail"` + `"I'll describe changes below"` |
| **Priority** | "Priority" | `"P00"` + `"P0"` + `"P1"` + `"P2"` |

**Always include** an "I'll [verb] below" option when free-text is acceptable.

---

## Step 1 — Gather inputs (AskUserQuestion only)

Batch in **one** call (up to 4 questions):

1. **"What's the feature you're writing the PRD for?"** — header "Feature". If a PRD doc was already created in this session, offer it as the first option.
2. **"Which source materials do you have?"** — header "Materials", multiSelect: Platform PRD (Lark URL) / Tech Design Doc (Lark URL) / Existing API docs (web URLs) / None — I'll describe it.
3. **"What type of feature is this?"** — header "Feature type": API Feature Parity / API Capability Enhancement / API First / Developer Experience. *(Fills the Feature Type line. No emoji on the title, whatever the type.)*
4. **"Which TikTok experience are we aligning to?"** — header "TT experience": TikTok One (TTO) / TikTok Ads Manager / TikTok Native App / None — N/A. **Never assume.**

**Follow-up call:** request the actual URLs via the **"Free-text input"** template, one question per material.

## Step 1.5 — Fetch all sources

Run the fetch ladder on every URL. For Lark docs, also fetch comments. Then report gaps explicitly and resolve each (paste / skip / retry) before Step 2.

---

## Step 2 — Analyze and present a plan

Present in text (no question yet):
1. One-line summary of what we're building and why.
2. Endpoints affected — name, request change, response change.
3. **Cascade checklist** — for every new field/enum/changed gate:
   - Sibling endpoints that should get the same treatment?
   - **Availability/eligibility notes already embedded in the response table** — "only available for Business Accounts", "requires at least 100 followers", etc. These are developer-visible contract text and are easy to miss; audit every row, not just the field you came for.
   - Enum mapping pages needing updates?
   - Upstream dependencies (auth scopes, account config, tag interfaces)?
   - Logic alignment (calculations, lookback windows, definitions)?
   - **Parity means same rules** — list eligibility constraints, edge cases, and error behaviors explicitly.
4. Open questions from doc body **and comments**.
5. Anything factually broken you noticed in the live doc (typos in field descriptions, contradictory timelines between sources). Winnie wants these; fold them into the doc-ticket scope.

**Then sign off** via the **"Sign-off"** template.

## Step 3 — Confirm user flow

AskUserQuestion with the **"Source selection (flow)"** template — including a `"Not needed"` option. Many API-side realignments have no developer-facing flow change; don't manufacture one.

---

## Step 4 — Write the PRD draft (section by section, interactive)

Read the existing Lark doc first.

**Do NOT write all sections at once.** One section at a time → present → AskUserQuestion → revise → next. Push only after all sections are approved.

### Before writing
- **No H1 repeating the doc title** — the Lark title is the heading. Start the body at `# 2. Summary`.
- **No emoji on the title** — do NOT add 🟢 or any other emoji to the Lark doc title or Meego workitem names, even for API Feature Parity features.

### Attention marker rule
Wrap anything Winnie must follow up on in `⚠️⚠️⚠️ … ⚠️⚠️⚠️`: TBDs, unconfirmed numbers, screenshot/whiteboard placeholders, open questions, "to be added manually".
> ⚠️ Never use `==text==` — it renders as literal `==`. Emoji markers only.

### Section sequence

1. **Summary** — **1–2 sentences, hard cap.** What, who, why, parity goal. Winnie has asked to tighten this when it ran to three. It is auto-extracted verbatim as the description on *both* Meego workitems, so it must stand alone.

2. **Basic Info — Feature Type**
   - Feature type from Step 1 Q3.
   - **No Effort line** (the template still shows one — drop it).
   - Priority: ask via the **"Priority"** template, then apply § Priority alignment rule.
   - **Change Log**: first row `| <today MM.DD.YYYY> | Initial Draft | @Winnie Lin |`, plus a new row for every push (see § Change Log rule).

3. **Background**
   - Problem Statement — what's missing vs the target experience, and the impact. Quantify blast radius where the sources let you.
   - Proposed Solution — what we're adding; note logic aligns with the target TT experience.
   - **"Experience on [TikTok X]"** — use Winnie's Step 1 Q4 answer *exactly*. If she said N/A, write one line saying so; do not invent a surface.
   - User Flow — numbered API call sequence, or omit if Step 3 said not needed.

4. **Success Metrics** — Goals → Success Metrics table. **1–2 rows max.**

5. **Requirements**
   - Scope: in/out bullets. Out-of-scope items still get ⚠️ markers when they're unresolved rather than decided.
   - API Change per endpoint — **each endpoint is its own review loop**.
     - **Existing endpoint:** pull the full current spec via the fetch ladder and reproduce it completely — every request parameter, every response field. Do not abbreviate. Mark new/changed rows in the status column.
     - **New endpoint:** `⚠️⚠️⚠️ [Winnie to fill in full schema] ⚠️⚠️⚠️` — invent nothing.
     - Full URL per Hard Stop #3. Close with `Note: Detailed field spec will be added as Lark sheet.`
     - Call out sibling-endpoint and enum-mapping impacts per field.

6. **Product Launch Plan** — **do not touch.** Skip in the review loop.
7. **Appendix** — **do not touch.** Skip in the review loop.

---

## Table format

*(Hard Stops #7 and #8. Get this right the first time — it has been corrected in every PRD so far.)*

**Response / parameter tables use one column per nesting level**, so the data structure is readable at a glance instead of encoded in dotted paths or indentation:

| L1 Field | L2 Field | L3 Field | L4 Field | Data Type | Description | Required permission | Status |
|---|---|---|---|---|---|---|---|
| `data` | | | | object | Return data | | — |
| | `metrics` | | | object[] | All requested daily metrics. … | | — |
| | | `audience_activity` | | object[] | Hourly follower activity. …**Note**: … | `user.insights` | **Changed** — … |
| | | | `hour` | string | The specific hour in the day. | | — |

- Use as many L-columns as the endpoint actually nests (L1–L3 is fine for a shallow one); keep them contiguous starting at L1.
- Put the field name in the column matching its depth and leave shallower columns blank — do not repeat parents.
- **Keep every column the source doc has** (e.g. `Required permission`), plus a trailing status column — `Status` or `Change` — marking `New` / `Changed` / `—`, with a short reason on changed rows.
- **Descriptions are verbatim** (Hard Stop #8): the full text including `Note:`, `Example:`, `Data source:`, `Updated in:`, and enum value lists. In XML, one `<p>` per line rather than `<br>`.
- Flat request-parameter tables (no nesting) stay `Parameter | Type | Required | Description`.

---

## Priority alignment rule

The PRD carries priority in two places: **§ 1 Meego Configuration → Project Priority** (which is what `prd2meego` actually sends to Meego) and the **Basic Info → Feature Type** line (which is what humans read). They have shipped out of sync — a Project created at P0 while the doc said P1.

So: after Winnie answers the Priority question, compare it to the § 1 cell.
- **Match** → write it into Basic Info, done.
- **Mismatch** → ask which one wins via AskUserQuestion. Only edit the § 1 Project Priority cell if she says to; that answer is the sole exception to Hard Stop #1, and it earns its own Change Log row.

---

## New field / endpoint naming rule

**Never name a new field or endpoint yourself.**

**Step A — Placeholder first.** `⚠️⚠️⚠️ [field name TBD — to be confirmed by Winnie] ⚠️⚠️⚠️`; keep drafting and come back.

**Step B — Ask the name** via AskUserQuestion with an `"I'll type it below"` option. Applies even when the name seems obvious from the feature description or the TTAM UI.

**Step C — Ask the type and shape.** *"Is `[field]` a primitive (string / int / bool / enum) or an object with sub-fields? If object, describe each sub-field (name, type, required-ness, constraints)."* Never assume a field is primitive because it ends in `_id`. Object fields need their own L-column rows.

If Winnie parks the field as a later-phase (P2) follow-up, record it under **Out of scope** with the name she used and `⚠️⚠️⚠️ type and enum values TBD ⚠️⚠️⚠️` — don't drop it, and don't spec it.

---

## Change Log rule

**Every push adds a row before pushing:**

| Date | Description | By |
|---|---|---|
| MM.DD.YYYY | Brief description of what changed | @Winnie Lin |

Date `MM.DD.YYYY`; description never blank (e.g. "First full draft — Summary through Requirements", "Created Meego workitems and linked them in Relevant Links"); By is always `@Winnie Lin`, even when Claude made the change. The `Initial Draft` row is never deleted.

---

## Push to Lark

**Pre-push checklist:**
- [ ] **§ 1 Meego Configuration unchanged** (→ Hard Stop #1), except an explicitly approved Project Priority edit.
- [ ] Every TBD, placeholder, and open question wrapped in `⚠️⚠️⚠️ … ⚠️⚠️⚠️`; no bare "TBD"/"pending"/"screenshot".
- [ ] Change Log has a new row for this push.
- [ ] All new field/endpoint names confirmed by Winnie.
- [ ] Tables use L1–Ln columns with verbatim descriptions (Hard Stops #7, #8).

**Assemble the whole doc as XML** — not markdown. Markdown flattens `<cite>` @mentions to plain text, and Basic Info needs real mentions.

```bash
cd '/Users/bytedance/Desktop/Ads PRD Workspace'
cat prd_draft_<feature>.xml | lark-cli docs +update \
  --doc <doc-token> --command overwrite --content -
```

- `--doc-format xml` is the default. **`--markdown`, `--mode`, and `--api-version` are dead flags** on `docs +update` — v2 rejects all three. (Older guidance showing `--markdown @file --mode overwrite` is wrong.)
- stdin (`--content -`) is simplest; `--content @file` needs a CWD-relative path.
- Targeted edits: `--command str_replace --pattern … --content …`, or `block_replace` / `block_insert_after` / `block_delete` with `--block-id` from `docs +fetch --detail with-ids`.
- A benign `degrade_code=1011 "no document changes"` warning can appear on a successful write — verify with `docs +fetch`, don't retry blindly.

**XML building blocks:** `<title>`, `<h1>`–`<h4>`, `<p>`, `<b>`, `<i>`, `<code>`, `<ul>/<li>`, `<ol><li seq="1">`, `<blockquote>`, `<hr/>`, `<a href>`, `<table><thead><tr><th background-color="light-gray">`, `<callout emoji="⚠️" background-color="light-red" border-color="red">`, and `<cite type="user" user-id="ou_…"></cite>`. Escape `&`→`&amp;`, `<`→`&lt;`, `>`→`&gt;` in cell text.

**Known open_ids:** Winnie Lin `ou_36af777e1a4474c3ebf223e2bb77a38b` · SangHa Park `ou_d3559ed314fdefc60b6aadd777153df5` · Juan Shishido `ou_89d1542fad2d667472ac6f535fef57c6`. Others: `lark-cli contact +search --query "<name>"`.

**Images:** `docs +media-download` returns `ok: true` but writes no file for images embedded in a docx. Don't burn turns — tell Winnie to copy-paste the image.

---

## Step 5 — Self-audit (run before debrief)

Re-read the assembled draft for each check; don't rely on memory. Fix and re-check before Step 6.

Mechanical checks worth scripting:
```bash
grep -nE '(GET|POST|DELETE|PUT) /open_api' draft.xml   # Hard Stop #3 — must be empty
grep -n '==' draft.xml                                  # highlight syntax — must be empty
grep -noE '.{60}(TBD|pending|to be confirmed).{60}' draft.xml   # each hit must sit inside ⚠️⚠️⚠️
grep -c '⚠️⚠️⚠️' draft.xml                              # even number (paired)
grep -nE '^<h1>' draft.xml                              # all 8 sections present
```

Then by eye:
- **#1** § 1 values identical to the Lark doc's.
- **#2** every parameter/endpoint name either exists in the public docs or was approved by Winnie this session.
- **#6** no internal tech-design names presented as developer-facing fields.
- **#7/#8** tables use L-columns; descriptions are full source text, not summaries.
- Product Launch Plan and Appendix byte-identical to the template.
- Cascade items from Step 2 are addressed or ⚠️-flagged.

Then run the pre-push checklist, push, and verify:
```bash
lark-cli docs +fetch --doc <token>   # confirm title, mention count, section count, marker count
```

---

## Step 6 — Debrief

Tell Winnie: what's done · what needs manual additions (Lark sheets, whiteboards, screenshots) · open questions inline as `[Question for Winnie: …]` · cascade items to verify.

End with AskUserQuestion: `["Looks good", "Revise section X", "Add another endpoint", "Run prd2meego"]`.

**Creating the Meego workitems is the `prd2meego` skill's Step 5 — follow it, not `prd2meego <url>` alone.** That command shells out to `bytedcli`, whose Meego auth expires independently and has failed at this exact step in most recent sessions. The working path is `prd2meego --dry-run` to build the payloads, then `meegle` to submit them.
