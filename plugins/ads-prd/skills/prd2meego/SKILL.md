---
name: prd2meego
description: >
  End-to-end workflow for creating a new Lark PRD document and linked Meego workitems (TTMP Project + Requirement/Story) for the Ads API team. Use this skill whenever Winnie asks to create a new PRD, start a new feature doc, write up a new API feature, or mentions "prd2meego", "Meego workitem", "Lark PRD", or "create a PRD for". Walk through the steps one at a time — don't skip ahead without Winnie's input.
---

# Ads PRD → Meego Workflow

Walk through these steps **one at a time**. Wait for Winnie at each handoff point.

> **Tooling summary (read first).** Two CLIs do the work, and neither is the one this skill used to name:
> - **Lark docs** → `lark-cli` (v2). `bytedcli lark docs update` is dead — it forwards flags v2 rejects.
> - **Meego workitems** → `meegle`. `prd2meego` shells out to `bytedcli`, whose Meego auth expires on its own schedule and needs an interactive login an agent cannot drive. Use `prd2meego --dry-run` to *build* the payloads and `meegle` to *submit* them (Step 5).

---

## Step 0 — Auth pre-check (cheap, do it before anything else)

```bash
lark-cli auth status | head -20
meegle auth status --format json
```

- `lark-cli` → needs `identities.user.available: true`. A `needs_refresh` token status is fine; it auto-refreshes on the next user API call.
- `meegle` → needs `"authenticated": true`. Note `expires_in_minutes`; if it's short, re-auth before Step 5 rather than mid-flight.

If `meegle` is logged out, it supports a device-code flow you can drive end to end (unlike `bytedcli meego login`, which is interactive-only):

```bash
meegle auth login --device-code --phase init --host meego.larkoffice.com --format json
# send Winnie verification_uri_complete (QR: lark-cli auth qrcode "<url>" -o qr.png), then:
meegle auth login --device-code --phase poll --client-id <client_id> --device-code-value <device_code> --format json
```

---

## Step 1 — Get the feature title

If Winnie hasn't already given you the title, ask for it. Format: `[Organic API] FeatureName` or `[PRD] FeatureName - Short tagline`.

**No emoji on the title** — do NOT add 🟢 or any other emoji to the Lark doc title or Meego workitem names, even for API Feature Parity features. `prd2meego` copies the title verbatim into both workitem names, so whatever is on the doc is what lands in Meego.

---

## Step 2 — Create the Lark doc

```bash
lark-cli api POST /open-apis/docx/v1/documents \
  --data '{"folder_token":"EYK5fyKQTlbJz9dTbIMcovRinhd","title":"<title from Step 1>"}'
```

Returns `data.document.document_id`. The folder is Winnie's **Ads PRD** folder.

**Do not construct the doc URL by hand.** The tenant is US-hosted — the canonical URL is `https://bytedance.us.larkoffice.com/docx/<document_id>`, and every `docs +update` / `docs +fetch` response echoes `data.document.url`. Read it from there.

Then populate the template (Step 3 builds the content).

---

## Step 3 — Populate the doc: template + Basic Info, in one push

Assemble the whole doc as **XML** in a local file, then overwrite in one call. Do not push markdown: `<cite>` @mentions flatten to plain text, and the Basic Info tables need real mentions.

```bash
cd '/Users/bytedance/Desktop/Ads PRD Workspace'
cat prd_draft_<feature>.xml | lark-cli docs +update \
  --doc <document_id> --command overwrite --content -
```

- `--doc-format xml` is the default — don't pass `--markdown`, `--mode`, or `--api-version`; v2 rejects all three.
- stdin (`--content -`) is simplest; `--content @file` only accepts a path relative to CWD.
- Source template: `/Users/bytedance/Desktop/Ads PRD Workspace/PRD_TEMPLATE_LARK.md` (markdown — convert its structure to XML).
- A `degrade_code=1011 "no document changes"` warning can appear even on a successful write. Verify with `lark-cli docs +fetch --doc <id>` rather than retrying blindly.

### XML building blocks

```xml
<title>[Organic API] Feature Name</title>
<h1>2. Summary</h1>
<p>…</p>
<table>
<thead><tr><th background-color="light-gray">Field</th><th background-color="light-gray">Value</th></tr></thead>
<tbody><tr><td>Space</td><td>ttmp</td></tr></tbody>
</table>
<callout emoji="📌" background-color="light-yellow" border-color="yellow"><p>…</p></callout>
<cite type="user" user-id="ou_…"></cite>
```

Escape `&` → `&amp;`, `>` → `&gt;` inside cell text. Multi-line cells use several `<p>` elements, not `<br>`.

### Known open_ids for the Basic Info mentions

| Person | open_id |
|---|---|
| Winnie Lin | `ou_36af777e1a4474c3ebf223e2bb77a38b` |
| SangHa Park | `ou_d3559ed314fdefc60b6aadd777153df5` |
| Juan Shishido | `ou_89d1542fad2d667472ac6f535fef57c6` |

Resolve anyone else with `lark-cli contact +search --query "<name>"`.

### Standard Basic Info format

**Meego Configuration** (§1) — Space `ttmp`, PM `winnie.lin@bytedance.com`, PRD URL = this doc's own URL, Project Priority (ask — P00/P0/P1/P2), Portfolio `Organic APIs`, Project Template `TTMP Project`, Technical Teams `MAPI`, Requirement Template `Ads Interface`. **Leave Project Name and Requirement Name blank** — `prd2meego` fills them from the doc title.

**Change Log** (3-col: Date | Description | By) — first row `MM.DD.YYYY | Initial Draft | @Winnie Lin`. Add a row on **every** subsequent push, including the one that links the Meego tickets.

**Relevant Links** (3-col: | Links | POC) — Link to Meego ticket (PM) / Legal Ticket (Legal) / DCC Legal Ticket / Tech Design (RD) / Comm Doc (PSO) / SLC Discussions/Proposal / API Doc Ticket / References.

**Authors / Core Team** — two separate tables:

| Team | Role | POC |
|---|---|---|
| **TT4B** | PM | @Winnie Lin |
| | PSO | @SangHa Park |
| | DE / SE / RD | (blank) |

| Team | Role | POC |
|---|---|---|
| **TTO** | PM | @Juan Shishido |
| | RD | (blank) |
| **Meego** / **Legal review** | | (blank) |

**Feature Type** — API Feature Parity / API Capability Enhancement / Developer Experience / etc., plus `**Priority: P0**`. No Effort line.

---

## Step 4 — Hand off to Winnie

Share the doc URL. Ask her to fill in Summary, Background, Success Metrics, Requirements. **Stop here and wait.**

Summary matters most: `prd2meego` auto-extracts it as the `description` on *both* workitems.

*(If she wants Claude to draft the content instead, that's the `write-prd` skill — run it, then come back to Step 5.)*

---

## Step 5 — Create the Meego workitems

### 5a. Build the payloads

```bash
prd2meego --dry-run "<lark-doc-url>"
```

This prints two JSON blocks — `=== Project payload (TTMP_Project) ===` and `=== Requirement payload (story) ===`. **`--dry-run` resolves the PM from local cache and makes no network calls, so it works even when `bytedcli` auth is completely dead.** That is the whole reason this two-tool split works.

Sanity-check the output before submitting: title (no emoji — not even for parity), `description` = the Summary, `field_0774ad` = clean doc URL (not `[text](url)` markdown), `field_181529` present.

Split the blocks into `meegle`-shaped params files:

```bash
python3 - <<'PY'
import json, pathlib, subprocess
url = "<lark-doc-url>"
txt = subprocess.run(["prd2meego","--dry-run",url], capture_output=True, text=True).stdout
head, tail = txt.split('=== Requirement payload')
proj = json.loads(head.split('===',2)[-1].strip())
req  = json.loads(tail.split('===',1)[-1].strip())
pathlib.Path('proj.json').write_text(json.dumps({"fields":proj}, ensure_ascii=False))
pathlib.Path('req.json').write_text(json.dumps({"fields":req},  ensure_ascii=False))
print('project fields:', len(proj), '| requirement fields:', len(req))
PY
```

### 5b. Create the Project, then the linked Requirement

**`--params` does NOT work on `workitem create`.** Passing `--params "$(cat proj.json)"` fails with `required flag(s) "fields" not set` — the JSON's top-level `fields` key no longer merges into the `--fields` flag. Pass **each field object as its own repeated `--fields` flag** instead:

```bash
python3 - <<'EOF'
import json, subprocess
fields = json.load(open('proj.json'))['fields']          # req.json for the story
args = ["meegle","workitem","create","--project-key","60c311455b37c203002327c6",
        "--work-item-type","610a4c1b8a3513ff598bdb85"]   # "story" for the Requirement
for f in fields:
    args += ["--fields", json.dumps(f, ensure_ascii=False)]
r = subprocess.run(args, capture_output=True, text=True)
print(r.returncode, r.stdout or r.stderr)
EOF
# → {"url": "...", "work_item_id": 7366991129}
```

Append `"--dry-run"` to `args` first if you want to see the normalized request before it hits the backend.

Set `field_65a611` in `req.json` to the Project's `work_item_id` (the dry-run leaves it `"0"`), then run the same snippet against `req.json` with `--work-item-type story`.

### 5c. Verify

```bash
meegle workitem get --project-key 60c311455b37c203002327c6 \
  --work-item-id <story_id> \
  --fields 'field_65a611,field_181529,field_0774ad,field_a1f49a'
```

**You must pass `--fields`** — without it every `work_item_fields` entry comes back `null` and the create looks like it silently dropped everything. The item name lives at `work_item_attribute.work_item_name`, not `.name`.

**The returned entries are keyed `key` / `name` / `value` — not `field_key` / `field_value`.** Parsing for `field_key` prints a column of `None = null`, which looks exactly like a create that dropped every field. Note the asymmetry: `create` takes `field_key`/`field_value`, `get` returns `key`/`name`/`value`. Read them like this:

```bash
meegle workitem get --project-key 60c311455b37c203002327c6 \
  --work-item-id <story_id> \
  --fields 'field_65a611,field_181529,field_0774ad,field_a1f49a' \
  | python3 -c "
import json,sys
for f in json.load(sys.stdin)['work_item_fields']:
    print(f['key'], f['name'], '=', json.dumps(f['value'], ensure_ascii=False))
"
```

Expect: Project → the new project id/name, Source of Requirements → `Core Infra-API`, Prd Link → the doc URL, Target Business Area → `API`.

### 5d. Link the tickets back into the PRD

Fill the **Link to Meego ticket** row in Relevant Links with both URLs, add a Change Log row ("Created Meego workitems and linked them in Relevant Links"), and push (Step 3's command). Then share both URLs with Winnie.

---

## Reference

**Keys**

| Thing | Value |
|---|---|
| Space `ttmp` project-key | `60c311455b37c203002327c6` |
| TTMP Project work-item-type | `610a4c1b8a3513ff598bdb85` |
| Requirement work-item-type | `story` |
| Ads PRD Lark folder token | `EYK5fyKQTlbJz9dTbIMcovRinhd` |

**Fields**

| Key | Meaning |
|---|---|
| `field_65a611` | Project (link from story → project) |
| `field_0774ad` | Prd Link |
| `field_181529` | Source of Requirements — `khx41xemm` = Core Infra-API. **Required** for `story` in ttmp |
| `field_a1f49a` | Target Business Area — `91zyxxqaw` = API |

## Gotchas

- **Read the `bytedcli failed (exit 1)` cause before assuming auth.** `MEEGO_AUTH_REQUIRED` → needs interactive `bytedcli meego login`. `EHOSTUNREACH`/`ETIMEDOUT` on a `10.x.x.x` address → bytedcli is reaching an internal corp host that needs VPN. `meegle` talks to public `meego.larkoffice.com` and works through both.
- **Check for partial creation before retrying** a failed run, so you don't double-create:
  ```bash
  meegle workitem query --project-key 60c311455b37c203002327c6 \
    --mql "SELECT \`工作项id\`, \`任务名称\` FROM \`ttmp\`.\`需求\` WHERE \`任务名称\` LIKE '%<title fragment>%'"
  ```
  MQL labels are `工作项id` and `任务名称` — `工作项ID` and `名称` both error. Type names for the FROM clause come from `meegle workitem meta-types --project-key <key>`: story is `需求`, project is `TTMP_Project`.
- `meegle workitem get` has **no** `--work-item-type` flag (unlike `create`). `workitem query` is MQL-only — there is no `--work-item-ids`.
- `meegle workitem meta-fields` requires `--page-num 1` and returns the list under a top-level `list` key, not `data`.
- Comments on a **wiki** doc: `file_type: "wiki"` is invalid. Resolve the node first — `lark-cli api GET /open-apis/wiki/v2/spaces/get_node --params '{"token":"<wiki_token>"}'` — then query comments with the returned `obj_token` and `file_type: "docx"`.
- `docs +media-download` returns `ok: true` but writes no file for images embedded in a docx. Don't burn turns on it; ask Winnie to copy-paste the image.
- No socks5 proxy is needed. `lark-cli` v2 reaches CN, US, MY, and SG-hosted docs directly.

## Improving this workflow further

`prd2meego` itself still shells out to `bytedcli` for both `workitem create` and the PM email→id lookup. Patching those two call sites to use `meegle` would collapse Step 5 back into a single `prd2meego <url>` command. Until then, the dry-run-plus-meegle path above is the reliable route.
