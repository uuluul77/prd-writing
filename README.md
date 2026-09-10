# winnie-skills

Ads-team Claude Code plugins: PRD drafting and the Lark PRD → Meego workitem workflow.

## Install

```bash
claude plugin marketplace add https://github.com/uuluul77/prd-writing.git
claude plugin install ads-prd@winnie-skills
```

The marketplace is named `winnie-skills` (from `.claude-plugin/marketplace.json`), which is
why the install target is `ads-prd@winnie-skills` and not the repo name.

## What you get

| Command | Skill | Does |
|---|---|---|
| `/ads-prd:prd2meego` | `prd2meego` | Creates a Lark PRD doc, then the linked Meego TTMP Project + Requirement pair. Five steps, one at a time. |
| `/ads-prd:write-prd` | `write-prd` | Drafts an Ads API PRD for TikTok feature parity. |

## Prerequisites

Both CLIs authenticate separately — run the Step 0 pre-check in the skill before starting.

```bash
brew install lark-cli meegle
```

`prd2meego` (Step 5) is a separate Python script, not a brew package — ask Winnie for a copy
and put it on your `PATH`.

## Known gaps for non-Winnie users

This is v1.0.0, written for a single author. Before your first run:

- The skill addresses you as **Winnie** throughout and sets the Meego PM field to
  `winnie.lin@bytedance.com` — change it to your own.
- **Step 2** creates the doc in Lark folder token `EYK5fyKQTlbJz9dTbIMcovRinhd` (Winnie's Ads
  PRD folder). You need write access, or swap in your own folder token.
- **Step 3** reads `PRD_TEMPLATE_LARK.md` from a local Ads PRD Workspace folder that is not in
  this repo. Ask Winnie for the template.
