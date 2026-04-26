---
description: Audit the tracked Peec prompt set for self-flattering bias and recommend remediation — either new prompts (expand) or swap-in-place (optimize within current budget) or both (hybrid)
argument-hint: [--mode=expand|optimize|hybrid — default hybrid] [--limit=N dry-run on first N prompts] [--plan-cap=N override plan-tier prompt cap if unknown]
---

# /peec-bias-audit

Produces an **evidence-based audit of the tracked Peec prompt set**: does the current prompt set structurally flatter the tracked brand, and if so, how do we fix it? Output is a Markdown report plus a CSV of prioritized, Peec-ready prompt recommendations. Sized dynamically — not a fixed number.

Two remediation tracks, both run by default (hybrid mode) and both included in the outputs:

- **Expand** — add new prompts to close blind spots. Sized to available plan-tier headroom; tail labeled for next-tier unlock. This is the product-led-growth path (more prompts → higher Peec plan).
- **Optimize in place** — swap the lowest-marginal-value inflation-risk prompts out and swap the highest-priority recommended prompts in. No plan change required. Trades some historical signal continuity for corpus honesty. Right choice for customers at their tracking cap who can't upgrade yet.

The runbook surfaces both paths so the reader decides. `--mode=expand` suppresses the optimize output; `--mode=optimize` suppresses the expand tail; `--mode=hybrid` (default) shows both and annotates a decision matrix.

## Why this audit exists

A Peec ranking is real *within the tracked set*. The open question: **is the tracked set itself tilted toward the brand's strengths?** If prompts over-represent the brand's dominant language, channel, persona, or vertical, a high ranking partly measures prompt selection, not market position. This audit tests that hypothesis without apology and returns an additive remediation plan.

## The 8 bias dimensions

Every prompt in the tracked set is scored 0–100 per dimension (0 = no bias, 100 = maximally biased toward the tracked brand's strengths). Dimension rubrics Claude applies in Step 4:

1. **Brand-phrasing bias** — does the prompt wording echo the tracked brand's own marketing copy (product names, category labels the brand coined, internal jargon)? Higher score = more copy-echo; unbiased framing uses generic buyer language (neutral category names: "customer support platform", "helpdesk tool", "CRM").
2. **Channel/feature skew** — does the prompt focus on the channel(s) or feature(s) where the tracked brand is strongest (inferable from the brand's top-cited URLs and highest-visibility topics)? >60 if the prompt is exclusive to that channel/feature; ≤20 if the prompt is multi-channel or anchored on a channel/feature where the brand is weaker.
3. **Geography / language bias** — is the prompt in the brand's home language and anchored to its home geography? >60 if home-language-only + home-geography; ≤20 if the prompt targets a non-home language or a geography the brand serves less strongly. Global competitors are structurally disadvantaged in home-language-only prompts.
4. **Buyer-stage (funnel) bias** — BOFU comparison prompts ("best X for Y", "top tools for Z") favor whichever brand has the stronger SEO footprint on those patterns. >60 if pure BOFU; ≤20 if TOFU informational ("how does AI messaging work") or MOFU alternative-seeker ("alternatives to Competitor").
5. **Persona bias** — prompts phrased for roles where the tracked brand's content and positioning resonate strongest (often Marketing / CX for messaging tools; Engineering / IT for dev tools; etc.). >60 if clearly phrased for the brand's strongest persona; ≤20 if phrased for persona categories the brand under-addresses (e.g. IT / Procurement / Ops / CFO / Founder for a Marketing-focused brand).
6. **Industry vertical bias** — prompts anchored to verticals the tracked brand serves heavily. >60 if in those verticals; ≤20 if in adjacent verticals the brand serves less deeply.
7. **Competitor-framing gap** — is the prompt of the form "Brand vs [X]" or "alternatives to [Competitor]"? These are how real buyers search and where every brand is structurally exposed. Scoring here is inverted: 0 = prompt IS competitor-framed (good, not biased), 100 = prompt lacks any competitor framing (biased toward aggregate visibility rather than head-to-head truth).
8. **Temporal recency bias** — does the prompt use outdated wording (prior-year phrasing) or miss currently-live categories (voice agents, AI-first automation, agentic tooling, MCP-era integrations)? >60 if outdated; ≤20 if current.

### Overall bias score

Per-prompt overall = weighted mean across the 8 dimensions (default equal weights). Corpus-level overall = mean of per-prompt overall, plus a separate "coverage entropy" score = how evenly the corpus distributes across each dimension's categories. A high corpus-level bias score with low coverage entropy = classic self-flattering prompt set.

## Step 1 — Resolve project and own brand

No IDs are cached in this repo. Every run discovers the workspace fresh.

1. `list_projects()` → the full set of projects this Peec account can see.
   - If **exactly one** project: use it, log the name.
   - If **more than one**: present the list to the user and ask which project to audit. Don't guess.
2. `list_brands(project_id)` → the full brand roster. **Find the brand with `is_own=true`** — that's the audited own brand. If exactly zero or more than one brand has `is_own=true`, abort with a clear message ("No own brand flagged" / "Multiple own brands flagged — resolve in Peec UI before re-running").
3. In parallel with step 2, also call:
   - `list_topics(project_id)` → `id → name` map.
   - `list_tags(project_id)` → `id → name` map.
   - `list_prompts(project_id)` → full array of prompts with `id`, `text`, `topic_id`, `tag_ids`, `created_at`.
   - `list_models(project_id)` → `id → name` map (AI engines in rotation).

Cache all resolved maps in working memory. The report resolves every ID to a human name on output — no raw IDs in either the Markdown or the CSV (CSV may carry IDs only in `replaces_pattern` for swap-pair traceability if needed).

If `$ARGUMENTS` contains `--limit=N`, narrow the prompt array to the first N (ordered by `created_at` desc) for dry-run validation. Default: all.

## Step 2 — Pull per-prompt performance

Two Peec calls, run in parallel. Use the full available data window — from the earliest chat date the workspace has data for, through today.

1. `get_brand_report(project_id, start_date=<workspace_start>, end_date=<today>, brand_id=[own_brand_id], dimensions=["prompt_id"])` → per-prompt own-brand metrics (visibility, SoV, mention_count, sentiment, position).
2. `get_brand_report(project_id, start_date=<workspace_start>, end_date=<today>, dimensions=["prompt_id","brand_id"])` → per-prompt × per-brand matrix. Used to establish which competitor *would* win a given prompt if the own brand weren't in the set — a direct signal of whether a prompt's own-brand win is narrow or dominant.

**Resolving the workspace start date:** if the project exposes a workspace-metadata tool, use its documented earliest-data date. Otherwise call `list_chats(project_id)` once, sorted ascending by `created_at`, and take the earliest chat date. Fall back to "30 days ago" if neither resolves.

**Known issue:** Peec transport occasionally drops mid-call. Retry once with 1s back-off before failing. If the dimension-keyed brand report is too large to complete in one call, iterate in batches of 30 prompt_ids and merge.

**Data-window caveat:** if `today - workspace_start < 7` days, add a report-level caveat that the window is shorter than a full competitive cycle.

## Step 3 — Classify each prompt (native LLM pass)

For each prompt (or first N under `--limit`), produce a JSON object:

```json
{
  "prompt_id": "pr_...",
  "text": "<prompt text>",
  "topic": "<resolved topic name>",
  "tags": ["<tag names>"],
  "scores": {
    "brand_phrasing": 0-100,
    "channel_skew": 0-100,
    "geo_language": 0-100,
    "buyer_stage": 0-100,
    "persona": 0-100,
    "industry": 0-100,
    "competitor_framing_gap": 0-100,
    "temporal": 0-100
  },
  "overall_bias": 0-100,
  "dominant_dimension": "<highest-scoring dimension name>",
  "rationale": "<1 sentence anchored in the prompt text>",
  "own_brand_visibility": 0.0-1.0,
  "own_brand_sov": 0.0-1.0,
  "own_brand_position": <number>,
  "top_competitor_in_prompt": "<brand name or null>"
}
```

Apply the dimension rubrics from the section above. Ground the rationale in observable wording — never invent facts the prompt doesn't show. If a dimension is genuinely N/A for a prompt, score it 50 (neutral) rather than 0 or 100.

**Inferring the brand's strengths for rubrics 2, 3, 5, 6.** The rubrics reference "channels where the brand is strong", "the brand's home language", "the brand's strongest persona", and "verticals the brand serves heavily". Infer these from the workspace itself — not from assumptions:

- **Home language** — the dominant language of the existing prompt set (detect from prompt text).
- **Strongest channel/feature** — look at the own brand's top-cited URLs via `get_url_report` and the topics with highest own-brand visibility in Step 2. Channels/features appearing there are the strengths.
- **Strongest persona and verticals** — infer from prompt wording and topic names in the existing set. If uncertain, score dimension 50 (neutral) and note in the rationale.

Hold all per-prompt classifications in memory. Write nothing to disk until Step 8.

## Step 4 — Flag inflation-risk prompts

A prompt is **inflation-risk** if ALL of the following are true:

- `own_brand_visibility > 0.4` (the brand wins the prompt convincingly), AND
- `overall_bias > 60` (prompt framing is self-flattering), AND
- at least one dimension score ≥ 70 (a specific, named bias driver, not just aggregate).

These are prompts whose own-brand wins are least representative of real competitive dynamics. Count them; list them in the report with the dominant bias dimension and rationale.

Additionally compute an aggregate `inflation_lift` estimate: `mean(own_brand_sov over inflation-risk prompts) - mean(own_brand_sov over non-inflation-risk prompts)`. Sign + magnitude indicate how much the inflation-risk subset elevates the reported SoV.

## Step 5 — Identify blind spots

For each of the 8 dimensions, compute coverage entropy across the corpus. A blind spot exists when one category dominates the dimension with >70% share while legitimate other categories are <10%. Example shapes a typical run might surface:

- **Geography:** one language ≈100%, other relevant languages ≈0% → blind spot on non-home-language tracking.
- **Channel:** one channel >50%, peer channels <10% each → blind spot on the under-tracked channels.
- **Buyer stage:** BOFU >60%, TOFU <15%, alternative-seeker <10% → blind spot on upper-funnel and competitor-branded searches.
- **Persona:** one persona >70%, others <10% → blind spot on the under-tracked personas.
- **Industry:** top-3 verticals capture >80% of prompts, long tail missing.
- **Competitor-framing:** zero or near-zero "Brand vs X" or "alternatives to X" prompts → blind spot on head-to-head tracking.
- **Temporal:** <5 prompts reference current-year categories (voice agents, AI-first, MCP, agentic) → blind spot on forward-looking queries.
- **Brand-phrasing:** >40% of prompts use the brand's own terminology → not a blind spot per se but a *corpus skew* to call out.

Record each blind spot as a structured finding: dimension, shortfall description, severity (0-100 based on share gap), and a count of how many new prompts would remediate the gap meaningfully (per-dimension target: raise the under-covered category to ≥15% corpus share).

## Step 6 — Size the recommendation set dynamically

The number of recommended new prompts is **derived per run**, not fixed. Compute:

```
weak_prompts           = count(inflation-risk prompts from Step 4)
blind_spot_gap_sum     = sum of remediation prompt counts across dimensions (Step 5)
raw_recommended_count  = weak_prompts + blind_spot_gap_sum
```

Then resolve plan headroom:

1. If `$ARGUMENTS` contains `--plan-cap=N`, use N as the total tracked-prompts cap.
2. Else try to infer from a safe source in this order: check if Peec MCP exposes a `get_workspace` / `get_plan` tool in the current session's tool list; if not, check docs.peec.ai via WebFetch for the public plan matrix; if still unknown, use `plan_cap = unknown`.

Compute:

```
current_prompts = len(list_prompts result)
headroom        = plan_cap - current_prompts   (or null if unknown)
```

Recommendation set size for this run:

- If `headroom` is a number: `fit_now = min(raw_recommended_count, headroom)`; `upgrade_tail = max(0, raw_recommended_count - headroom)`.
- If `headroom` is null: `fit_now = raw_recommended_count` with a report footnote stating "plan headroom unresolved — confirm with Peec before adding more than X prompts," and the upgrade tail is reported as "unknown (confirm cap)."

The report must state the sizing formula explicitly so readers can replicate it.

## Step 6b — Rank inflation-risk prompts for swap-out (optimize mode)

For the `optimize` and `hybrid` modes, the inflation-risk prompts from Step 4 are ranked by **marginal value** (how much *unique* signal each one contributes, low = safe to retire):

```
marginal_value = 0.5 * uniqueness_score          (0-100 — how dissimilar the prompt is vs. other tracked prompts on topic + wording)
               + 0.3 * recency_of_movement       (0-100 — W-o-W SoV delta magnitude; stable prompts contribute less new signal)
               + 0.2 * cross-model_variance      (0-100 — prompts where engines disagree are higher-signal)
```

Low marginal_value = safe retirement candidate. Never retire a prompt that is:

- the only tracking of its topic or tag (would orphan a dimension)
- one of fewer than 3 prompts in its topic (preserve topic coverage)
- the single top-citation prompt per topic (historical-continuity anchor)

Output the retirement-rank order; the swap pairs are formed in Step 7.

## Step 7 — Generate the recommended prompt list

For each blind spot AND each inflation-risk prompt, generate concrete Peec-ready replacement/augmentation prompts. Each recommended prompt is an object:

```json
{
  "prompt_text": "<ready to paste into Peec UI>",
  "topic_suggestion": "<existing topic name if fit; else a new topic name>",
  "language": "de|en|fr|nl|...",
  "persona": "marketing|cx|it|procurement|ops|cfo|founder",
  "buyer_stage": "tofu|mofu|bofu|alternative-seeker|competitor-vs",
  "channel": "whatsapp|voice|email|sms|rcs|web-chat|social-dm|multi|n/a",
  "industry": "retail|beauty|automotive|fintech|saas|healthcare|ecommerce|agency|enterprise|cross",
  "targets_dimension": "<which of the 8 dimensions this closes>",
  "priority_score": 0-100,
  "rationale": "<1 sentence>"
}
```

**Priority score formula:**

```
priority = 0.4 * dimension_severity        (Step 5, 0-100)
         + 0.3 * inflation_impact          (0-100 — own-brand SoV lift if this prompt replaces an inflation-risk one)
         + 0.2 * ease_of_execution         (100 = minimal new tracking setup; 0 = requires new topic/tag)
         + 0.1 * competitor_exposure       (100 = directly tests a tracked competitor; 0 = no competitor in prompt)
```

Sort the recommended list by priority descending. Top-N (= `fit_now`) are "add now"; remainder are "unlocks at next plan tier."

Guarantee variety: within the top-N, no single dimension should contribute more than 40% of the slots unless only one blind spot exists — maintains cross-dimensional balance.

### Swap pairing (optimize + hybrid modes)

Form `swap_capacity = min(count(safe-retirement inflation-risk prompts from Step 6b), top_priority_recommendations)`. Pair them greedily:

- Swap-in rank 1 (highest priority recommendation) ↔ Swap-out rank 1 (lowest marginal value retirement candidate).
- Continue until `swap_capacity` is exhausted.

Mark each swap-in prompt with `swap_in_candidate = yes` and its paired retirement target (topic or pattern reference, not raw ID, for the report; raw ID in the CSV). Mark each retirement target with a row in the report's retirement table.

Every prompt in the recommendation set stays in the CSV regardless of mode — the `tier` column distinguishes `swap_in` (would be swapped in under optimize/hybrid), `add_now` (would be added as expansion under expand/hybrid), and `next_plan_tier` (exceeds headroom under expand mode).

## Step 8 — Write outputs

Two files, timestamped with today's date (local timezone):

### A) `reports/prompt-bias-audit-<YYYY-MM-DD>.md`

**Presentation principles** (apply throughout — the goal is a report a reader can scan in under 60 seconds and present on-screen without reading aloud):

- **Tables over prose.** Every factual claim that repeats a shape (dimension N has current=X, why=Y, fix=Z) goes in a table, not a paragraph. Paragraphs are reserved for one-line framing sentences.
- **Lead every major section with a blockquote** (`> ...`) carrying the single most important sentence. The rest of the section backs it up in scannable form.
- **Severity emojis** 🟥 / 🟧 / 🟩 on every dimension heading and in every heatmap / critical-blind-spot table. Consistency matters — the reader learns the vocabulary once.
- **ASCII bar chart** for the bias dimension heatmap, rendered inside a fenced `plaintext`-style block using `█` characters scaled to the severity score (one `█` per ~2 severity points, right-aligned severity label after a two-space gap). Followed by the canonical table — chart for at-a-glance, table for detail.
- **`<details>` collapse for Methodology and Appendix.** The reader should see the findings first; methodology/appendix are expand-on-demand.
- **Horizontal rules (`---`)** between all top-level sections. No section headings without a rule above.
- **Per-dimension sections use an identical template** (see below). Do not improvise prose.
- **No raw IDs.** Resolve all `pr_…`, `kw_…`, `to_…` to human names before rendering.
- **Percentage-point vs. percent:** SoV / visibility deltas are always "+X pp" or "−X pp" (never "+X%"). Display ratios as percentages (×100) but show rate metrics (citation_rate, retrieval_rate) as-is.

**Section order (inverted vs. metric-first dashboards — findings → risk → recommendations):**

1. **Title + tagline blockquote** — one line. Always lead with the single most important calibration sentence (e.g. "Our reported X% SoV overstates our lead by N pp.").
2. **At-a-glance card** — a 4-column single-row table: data window · corpus size · overall bias score · plan headroom status. Use emoji column headers (📅 🏷️ 🎯 🏦).
3. **Scorecard** — 4-column table: Metric · Reported · Honest · Delta. Rows: Visibility · SoV · Avg position · Sentiment. Follow with a blockquote contextualizing the delta.
4. **🟥 Critical blind spots** — numbered 3-row table: `#` · Dimension (bold) · Status · Fix (`+N prompts`). Strictly the 🟥 dimensions.
5. **Two remediation tracks** — side-by-side comparison table (Track A column, Track B column) with rows: What · Plan change · Coverage impact · Time to value · Trade-off. Follow with a blockquote on composability.
6. **Bias dimension heatmap** — ASCII bar chart first, canonical table second. Table columns: Dimension · Dominant category · Coverage entropy · Severity (emoji + score) · Add (N).
7. **Inflation-risk prompts** — a top-of-section card table (Count · % of corpus · Inflation lift), a blockquote on the stacking pattern, then a "Representative pattern" table (Slice · Dominant dimensions · Why it inflates). No per-prompt listing in the report — enumeration lives in the CSV and the live-run appendix.
8. **Per-dimension findings** — 8 sections. **Use this template exactly**, nothing else:
   ```
   ### [emoji] [N]. [Dimension name] — severity [score]
   
   > **[One-sentence finding.]** [One-sentence context.]
   
   | | |
   |---|---|
   | **Current** | [short fact] |
   | **Why it matters** | [short reason] |
   | **Blind spot** | [what's not measured] |
   | **Fix** | +N prompts — [what shifts] |
   
   **Top 3 additions**
   
   1. [Prompt text] *(priority N)*
   2. [Prompt text] *(N)*
   3. [Prompt text] *(N)*
   
   ---
   ```
9. **Execution Roadmap** — Track A quartile table + Track B retirement pattern table + Track B swap-in pairing table (1:1 with ↔ column) + Net impact delta table + Decision matrix. Every Track B table has a blockquote trade-off note. Sub-sub-headers use 🚀 (Expand) / ⚖️ (Optimize in place).
10. **Methodology** — inside `<details><summary><strong>Methodology</strong> — …</summary>`. Contains the rubric table, sizing formula (fenced code block), and confidence caveats.
11. **Appendix** — inside `<details><summary><strong>Appendix</strong> — …</summary>`. The full per-prompt classification table.
12. **Footer** — single centered line with generation date and `#BuiltWithPeec`: `<p align="center"><em>Generated [date] · #BuiltWithPeec</em></p>`.

**Length budget.** The rendered report should feel ≤ 1 scroll per section. If a section exceeds ~25 rendered lines, tighten the prose first, then consider a `<details>` collapse for any sub-table that isn't primary.

### B) `reports/recommended-prompts-<YYYY-MM-DD>.csv`

Columns (header row):

```
priority,targets_dimension,topic_suggestion,language,persona,buyer_stage,channel,industry,prompt_text,rationale,tier,swap_in_candidate,replaces_pattern
```

- `tier` ∈ `{swap_in, add_now, next_plan_tier}`.
  - `swap_in` = top-N by priority paired 1:1 with a retired inflation-risk prompt (optimize + hybrid modes).
  - `add_now` = fits current plan headroom as new prompts (expand + hybrid modes when headroom allows).
  - `next_plan_tier` = exceeds headroom under expand mode.
- `swap_in_candidate` = `yes` for the top-N paired for swap; `no` otherwise.
- `replaces_pattern` = short descriptor of the retirement target (e.g. "home-language BOFU pricing comparison") for swap_in rows; empty otherwise.
- CSV-safe quoting: every field containing commas or quotes wrapped in double quotes with internal quotes escaped.
- One row per recommended prompt, sorted by priority descending, then `swap_in_candidate` yes before no.

### File collision

If either target file already exists for today's date, append `-v2`, `-v3`, etc. — do not overwrite.

## Presentation rules

- Never show raw IDs (`kw_…`, `pr_…`, `to_…`). Resolve everything to human-readable names via the Step 1 lookups.
- Never show raw JSON in the report (Appendix is a markdown table, not a blob).
- Metrics: visibility / share_of_voice / retrieved_percentage are 0–1 → display as percentages ×100. sentiment is 0–100 → display as-is. position is a rank → display as-is (lower is better).
- Inflation-lift and SoV deltas are in percentage *points*, not percentages — label accordingly ("+4 pp to SoV", not "+4%").
- Call out 0% visibility or 100% coverage-dominance explicitly when they appear; they're the most actionable findings.
- Do not narrate tool calls or classification progress. The final output is the two file paths plus a 2-sentence chat summary.

## Starting action

Silently execute Steps 1–8. Resolve the prompt cap (Step 6) via WebFetch to docs.peec.ai if MCP doesn't expose it. Write the two artifacts and report their paths plus a one-paragraph summary (overall bias score, recommended prompt count with formula, top 3 blind spots, honest-SoV estimate).
