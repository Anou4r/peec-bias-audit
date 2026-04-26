# Prompt-Bias Audit for Peec

*A Claude Code workflow that audits your Peec prompt set for self-flattering bias — and returns a prioritized, plan-tier-sized list of prompts to add, swap, or retire so your AI-visibility tracking measures the market, not your marketing copy.*

One slash command. No scripts, no API keys, no dependencies beyond the Peec MCP server. Clone, authorize, run.

---

## The problem this solves

Peec tells you how you rank across AI engines — **within the set of prompts you chose to track**. That's the catch. If the tracked set over-represents your home turf (your strongest language, your dominant channel, your favorite buyer persona, the verticals where you already win), a #1 ranking can partly reflect the prompts you picked rather than the market's real preference.

Every tracked prompt set drifts this way. Marketing picks prompts that sound like the category. Product picks prompts that showcase features. Growth picks the BOFU comparisons where SEO already converts. None of it is dishonest — it's just that **the tracking set inherits the author's blind spots**, and the #1 ranking inherits them too.

Left unchecked this costs you two things:

1. **Calibration.** You make GTM and product bets against a metric that overstates your lead in precisely the corners of the market where you're already strong.
2. **Discovery.** You never see the queries where you're losing because you're not tracking them. Voice AI, English-language, head-to-head "alternatives to X" — whichever dimension is your blind spot is also the one your competitors are quietly winning on Peec's engines right now.

`/peec-bias-audit` exists to surface that, quantify it, and return a fix.

---

## What the command does

One run produces two artifacts:

| Artifact | Purpose |
|---|---|
| `reports/prompt-bias-audit-<DATE>.md` | Readable audit — bias heatmap, scorecard (reported vs. honest estimate), critical blind spots, per-dimension findings, two remediation tracks, decision matrix |
| `reports/recommended-prompts-<DATE>.csv` | Peec-ready prompts with metadata (priority, topic, language, persona, buyer stage, channel, industry, swap-or-add tier) — paste directly into the Peec UI |

Under the hood, every prompt in your tracked set is scored on **8 bias dimensions** (brand-phrasing, channel skew, geography/language, buyer stage, persona, industry, competitor-framing, temporal recency). The runbook cross-references those scores against each prompt's per-engine performance from `get_brand_report` to flag **inflation-risk prompts** — prompts you win where the framing is structurally self-flattering. Per-dimension **coverage entropy** surfaces blind spots where one category (e.g. one language, one channel) dominates the corpus.

The fix comes in two flavors, both returned every run:

- **Expand** — add new prompts to close blind spots. Sized to your current plan-tier headroom; the tail is labeled "unlocks at next tier." This is the product-led-growth path: more honest tracking → more prompts → higher plan tier → better data.
- **Optimize in place** — swap the lowest-marginal-value inflation-risk prompts out and swap the highest-priority recommendations in, 1:1. No plan change required. Trades some W-o-W historical continuity on the retired prompts for corpus honesty.

A decision matrix in the report tells the reader which mode fits their situation. `--mode=expand`, `--mode=optimize`, or `--mode=hybrid` (default — both).

---

## Business impact

Three audiences, one artifact:

- **For Peec customers** — honest tracking produces calibrated GTM decisions. If your reported SoV overstates your lead by 3 pp because of corpus skew, every positioning bet you stack on it inherits that error. The audit replaces the vibes-based "are we really #1?" question with a reproducible answer, run quarterly.
- **For Peec itself** — the audit surfaces every customer's next plan-tier upgrade rationale, data-backed. The top-N recommendations fit the current cap; the tail is labeled as the next-tier unlock. It converts tracking-cap friction into a principled expansion conversation — that's a product-led-growth loop Peec can rerun with every customer, every quarter.
- **For the AI-visibility category** — this is a worked example of how to audit an LLM-visibility tracking set against its own blind spots. The 8-dimension rubric and the sizing formula are portable to any tool that lets you enumerate and score a prompt corpus.

**When to run it:** quarterly, or whenever you materially change the tracked set, or after a market shift (new category, new competitor, new channel).

---

## Try it yourself

```bash
git clone https://github.com/<your-fork>/peec-bias-audit
cd peec-bias-audit
claude                              # Claude Code launches with the project's .mcp.json
/mcp                                # authorize Peec (api.peec.ai/mcp) on first run — OAuth, no keys
/peec-bias-audit                    # default: --mode=hybrid — both expand + optimize tracks
/peec-bias-audit --mode=optimize    # swap-only: no plan change required
/peec-bias-audit --mode=expand      # add-only: requires plan-tier headroom
/peec-bias-audit --limit=10         # dry-run on the first 10 prompts to spot-check classification
/peec-bias-audit --plan-cap=300     # override the plan-tier prompt cap if not resolvable
```

The first `/peec-bias-audit` run:

1. calls `list_projects` — if you have multiple Peec projects it asks which to audit,
2. calls `list_brands` and picks the brand flagged `is_own=true` as the audited brand,
3. pulls every tracked prompt plus per-prompt performance via `get_brand_report` with `dimensions=["prompt_id", "brand_id"]`,
4. classifies every prompt on the 8 bias dimensions in-session (native Claude classification, no third-party LLM call),
5. writes the two artifacts to `reports/` with today's date.

No keys, no `.env`, no ID hardcoding. The command discovers your workspace on first run.

---

## Implementation notes

### Tools used

**Peec MCP** (`https://api.peec.ai/mcp`) — `list_projects`, `list_brands`, `list_topics`, `list_tags`, `list_prompts`, `list_models`, `list_chats`, `get_chat`, `get_url_content`, `get_brand_report` (dimension-keyed), `get_url_report`, `get_domain_report`, `get_actions`. Only Peec MCP — no other server required.

### What the command does well

- **Classifies all prompts natively in Claude Code** — no external LLM call, no API-key juggling, deterministic re-runs against the same data window.
- **Uses Peec's dimension-keyed `get_brand_report`** (dimensions `["prompt_id", "brand_id"]`) to establish per-prompt competitive matrices — surfacing which competitor *would* win a given prompt if your brand weren't in the set. That's what exposes inflation risk.
- **Sizes the recommendation set to plan-tier headroom** — not a round number, not a dashboard KPI. Top-N fits the current tier; the tail is labeled as the upgrade-unlock delta.
- **Returns two remediation tracks — expand and optimize — every run.** The decision matrix in the report tells you which mode fits your plan situation, so the audit stays useful whether or not an upgrade is viable right now.
- **Auto-discovers your project and own brand** on every run. No "edit this config file" setup step — clone and go.
- **Inverts the report order vs. a metrics dashboard.** Findings and risk come first, recommendations last. The reader's first question ("should I trust my #1 ranking?") is the report's first section.

### Known limitations documented in the runbook

- Peec MCP occasionally drops mid-call. The runbook retries once with 1s back-off before failing, and iterates dimension-keyed reports in batches of 30 prompt IDs if the response is too large.
- If the Peec MCP session doesn't expose a workspace/plan-cap tool, the command attempts to resolve the cap from docs.peec.ai; if still unresolved, it runs and flags "headroom unknown" rather than guessing.
- Classification is LLM-based and should be spot-checked on the first run — `--limit=10` exists exactly for that.

---

## Repo layout

```
.
├── .mcp.json                                  Peec MCP config (project scope)
├── .claude/
│   ├── commands/
│   │   └── peec-bias-audit.md                 /peec-bias-audit runbook
│   └── settings.local.json                    Tool allowlist (machine-local, gitignored)
├── reports/
│   ├── README.md                              Report format reference (section order, CSV schema)
│   ├── prompt-bias-audit-YYYY-MM-DD.md        Generated by each run
│   └── recommended-prompts-YYYY-MM-DD.csv     Generated by each run
├── .env.example                               Placeholder — OAuth only, no keys needed
├── .gitignore
├── LICENSE                                    MIT
└── README.md                                  This file
```

---

*Built with Claude Code and the Peec MCP server. Originally developed as a submission to the [Peec MCP Challenge](https://peec.ai/mcp-challenge) — Competitive Analysis category. `#BuiltWithPeec`*
