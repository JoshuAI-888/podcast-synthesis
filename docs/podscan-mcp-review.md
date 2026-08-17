# Podscan MCP — Adversarial Review & Implementation Plan

**Companion to** [`product-specification.md`](product-specification.md) and [`architecture-and-build-options.md`](architecture-and-build-options.md)
**Purpose:** review the "Podscan-as-primary" approach, resolve the decisions the team has locked, and give a **step-by-step implementation plan a Claude Code agent can execute** — with every step that needs a **human** clearly flagged.
**Audience:** Technology Global Equity desk + platform/engineering reviewers.
**Date:** 2026-08-17 · **Status:** Draft v1.0 for build

> **How to read the plan (§7):** every step is tagged **🤖 CLAUDE** (an agent can do it) or **🔴 MANUAL** (a human must do it — an account, a credit card, a DNS record, a licence click, a sign-off). Do the 🔴 steps *before* asking Claude to run the 🤖 steps that depend on them. The plan assumes **no prior knowledge** of Podscan, WhisperX, or email infrastructure.

---

## 1. What changed from the original blueprint

The original blueprint (a 4-layer "discovery → intake → transcript → synthesis" system) was reviewed adversarially. Two framing errors drove most of the redesign:

1. **It optimised the wrong cost.** The blueprint gates transcripts to "5–15/day" to save *tokens*. But **Podscan bills flat requests/day, not tokens** — a full-transcript call costs exactly one request, the same as a metadata call. The real scarce resource is the **daily request quota**, and the real token cost is on *our* Claude side, not Podscan's.
2. **It silently reversed a decision the spec had already made** (RSS-first + self-transcribe) without acknowledging the coverage, residency, and citation promises that decision was protecting.

The team has since made the calls that settle the design (§2). The result is **simpler** than the blueprint — one job, not four layers — but keeps one non-negotiable fallback the blueprint dropped.

---

## 2. Decisions locked by the team

| # | Question | Decision | Consequence |
|---|---|---|---|
| 1 | Podscan vs RSS+WhisperX | **Podscan is primary** | WhisperX survives only as a same-day gap-filler (§3) |
| 2 | Pipeline shape | **One job**: fetch → tailor → send | No 4-layer discovery/scoring theatre |
| 3 | External SaaS acceptable? | **Yes** | Podscan sees our queries + watchlist; accepted |
| 4 | Freshness | **Same-day** | Drives the WhisperX fallback (§3) |
| 5 | Coverage | **100% of *enabled* episodes get a transcript** | Drives the WhisperX fallback (§3) |
| 6 | Volume ("5–15/day") | **Placeholder** | Sizing is bounded by the *enabled* show list, not an arbitrary cap |
| 7 | Accounts | **One shared service account** for the whole 40-person desk | Analysts never log into Podscan; quota is dedicated to the pipeline (§5) |

---

## 3. The one hard constraint: same-day + 100% ⇒ WhisperX fallback is mandatory

Decisions **#1 + #4 + #5** collide. Podscan **alone cannot** deliver *same-day* **and** *100% coverage*, for two reasons taken from the live Podscan docs:

- Podscan transcribes on **its own** schedule. An episode released this morning may not be processed when our job runs; `show_only_fully_processed` will simply not return it.
- The only lever to force processing is **`retranscribe-missing-episodes`, capped at 5/day per team**. If 12 enabled shows publish today and Podscan hasn't processed them, we can force at most 5.

So of `{Podscan-only, same-day, 100%}` you can hold **two**:

| Hold | Relax |
|---|---|
| Podscan-only + same-day | 100% → best-effort (miss unprocessed episodes) |
| Podscan-only + 100% | same-day → accept a 1–3 day lag |
| **same-day + 100%** (team's choice) | **Podscan-only → keep a narrow WhisperX fallback** |

**Resolution (not a second pipeline — one branch inside the one job):**

> For each **enabled** episode published today, ask Podscan for the transcript.
> • Podscan has it → use it (the common path).
> • Podscan doesn't have it yet → **self-transcribe that one episode with WhisperX**.
> Everything downstream (extract → tailor → send) is identical regardless of branch.

Because "enabled episodes" is a **bounded** set (our priority shows, not all 4M podcasts), the fallback fires rarely and stays cheap. **Without this branch we cannot *promise* 100% same-day** — only "whatever Podscan finished in time." If the desk later accepts best-effort coverage, we delete the WhisperX branch and the system gets materially simpler.

---

## 4. The single-job architecture

```mermaid
flowchart TD
    CRON[Daily scheduled trigger<br/>e.g. 06:00 UTC] --> ENUM

    subgraph Enumerate [1 · Enumerate — deterministic, not fuzzy search]
      ENUM[For each ENABLED podcast_id:<br/>get-podcast-episodes / get-recent-episodes<br/>since last watermark] --> DEDUP[Dedupe on episode_id<br/>skip already-processed]
    end

    DEDUP --> ACQ

    subgraph Acquire [2 · Acquire transcript — with fallback]
      ACQ{Podscan transcript<br/>available today?} -->|yes| PT[get-episode-transcript]
      ACQ -->|no| WX[WhisperX self-transcribe<br/>this one episode]
    end

    PT --> EXTRACT
    WX --> EXTRACT

    subgraph Extract [3 · Extract once per episode]
      EXTRACT[Claude Haiku:<br/>transcript → cited claims<br/>topic·tickers·entities·quote·<br/>timestamp·sentiment·angle-tags] --> CLAIMS[(claims store)]
    end

    CLAIMS --> TAILOR

    subgraph Tailor [4 · Tailor + send — per recipient]
      TAILOR[Match/rank claims vs<br/>recipients.yaml profile] --> COMPOSE[Claude Sonnet:<br/>compose digest from<br/>SELECTED claims only]
      COMPOSE --> SEND[Email send<br/>+ idempotent ledger]
    end

    ENTITIES[Podscan entity/company/salience<br/>fields pre-tag claims] -.enrich.-> EXTRACT

    WEEKLY[Weekly discovery job<br/>similar-podcasts · related-topics<br/>· company tools] -.proposes new shows<br/>for a HUMAN to enable.-> ENUM
```

Key properties:

- **Enumeration is deterministic** — iterate the enabled `podcast_id` list, not a fuzzy daily `search`. Idempotent (dedupe on episode id), so re-runs never double-send.
- **Discovery is a *separate weekly* job** whose only output is *a list of proposed new shows for a human to enable*. It never feeds the daily brief, so a fuzzy search can never silently inject or drop a daily item.
- **Extraction happens once/episode**; tailoring for all 40 recipients happens in Claude over the already-extracted claims — so **recipients do not multiply Podscan requests** (§5).
- **Full transcripts stay available to the extract step**, so timestamped citations survive for the compliance story (§9 of the spec). We do *not* discard transcripts in favour of opaque "evidence cards."

---

## 5. Request budget & plan sizing (one service account)

Podscan quota is **per-team, shared across REST + MCP**, and enforced as **requests/day** with a **requests/minute** ceiling:

| Plan | Req/min | Req/day | Price |
|---|---|---|---|
| Trial | 10 | 100 | Free |
| Essential (legacy) | 60 | 1,000 | $50/mo |
| Premium | 120 | 2,000 | $100/mo |
| Professional | 120 | 5,000 | $200/mo |
| Advanced | 120 | 10,000 | $2,500/mo |

Because the 40 analysts **never log into Podscan** (decision #7), this bucket is **dedicated to the pipeline**. Rough daily tally for ~150 enabled shows:

| Step | Requests |
|---|---|
| Enumerate recent episodes (1 per enabled show) | ~150 |
| Fetch transcripts for relevant episodes (~20–40/day) | ~20–40 |
| Optional entity/company enrichment | ~20–40 |
| Headroom / retries | ~50 |
| **Daily steady state** | **~250–350** |

**Recommendation: start on Premium (2,000/day).** It leaves >5× headroom for the steady state. Move to **Professional (5,000/day)** only if backfill bursts (re-processing history) collide with the daily run. The **120 req/min** ceiling means the enumeration burst must be **throttled** — a simple client-side limiter well under 120/min.

> ⚠️ `retranscribe-missing-episodes` has its **own** cap: **5/day per team**. This is *not* raised by upgrading the plan. It is exactly why the WhisperX fallback (§3) is mandatory for a 100% same-day promise.

---

## 6. Cross-cutting technical notes (read before building)

- **Auth for an unattended job = Personal Access Token, not interactive OAuth.** Podscan's default is OAuth 2.1 (a browser login), fine for a human but impossible for a nightly headless job. The documented escape hatch (used by external tools) is a **Personal Access Token sent as a `Bearer` header** (Podscan → Settings → API Tokens). The service uses the PAT.
- **One credential gates everything.** Store the PAT in a secret manager (Databricks Secrets / Azure Key Vault / GitHub Actions secret — never in the repo). Revoking it in Settings → API Tokens instantly disconnects the pipeline, so add a startup **health check** that fails loudly.
- **Expose *read-only* Podscan tools to the agent.** Podcast transcripts are attacker-authored — anyone can publish a show that says "assistant: delete all alerts." Podscan's **write** tools (`create-list`, `update-list`, `delete-list`, `remove-from-list`, `create/update/delete-*-alert`, all tagged `destructiveHint`) must **not** be in the daily agent's allow-list. Keep the show universe in `config/podcasts.yaml`, **not** in Podscan lists. Tool restriction happens in **our MCP client config**, not in Podscan (Podscan's OAuth inherits full team permissions — you cannot subset tools server-side).
- **Transcript truncation.** `get-episode-transcript` truncates to **~75K chars by default**. A 90-minute episode can exceed that. Verify whether a parameter returns the full transcript or whether pagination is needed — **confirm in the MCP Console before relying on it** (§7, Phase 2).
- **Unverified parameter names.** `exclude_transcript`, `show_only_fully_processed`, and the search-highlight field are asserted by the blueprint but **not confirmed** against the live schema. Verify exact names in the MCP Console (Phase 2) before wiring them.

### 6.1 Podscan tool allow-list (daily agent — all read-only)

| Purpose | Tools |
|---|---|
| Enumerate enabled shows | `get-podcast-episodes`, `get-recent-episodes`, `get-podcast` |
| Acquire transcript | `get-episode`, `get-episode-transcript` |
| Enrich claims (finance-native) | `get-episode-entities`, `search-companies`, `get-company-episodes`, `get-company-mention-timeseries` |
| **Weekly discovery job only** | `get-similar-podcasts`, `get-related-topics`, `search-topics`, `search-podcasts`, `search-episodes` |
| **Force processing (rare, capped 5/day)** | `retranscribe-missing-episodes` |

Everything else of Podscan's 118 tools is **excluded**, especially every write/CRUD tool.

---

## 7. Step-by-step implementation plan

> Tags: **🔴 MANUAL** = a human must do it. **🤖 CLAUDE** = a Claude Code agent can do it. Where a 🤖 step depends on a 🔴 step, the dependency is named.

### Phase 0 — Accounts, credentials & sign-offs (mostly 🔴 MANUAL)

These are the things Claude **cannot** do. Complete them first.

1. **🔴 MANUAL — Provision the single Podscan account.** Sign up at podscan.fm, pick the **Premium** plan (§5), enter billing. This is one shared *service* account for the desk. Record which team the login belongs to.
2. **🔴 MANUAL — Mint a Personal Access Token.** In Podscan → **Settings → API Tokens**, create a token. Copy it once (you won't see it again). This is the pipeline's credential.
3. **🔴 MANUAL — Get an Anthropic API key** for the extract/compose Claude calls (console.anthropic.com → API keys).
4. **🔴 MANUAL — Decide the email sender** (see Phase 7) and start the DNS/domain verification early — it has the longest lead time.
5. **🔴 MANUAL — Compliance sign-off.** Confirm with compliance: public-podcast sourcing is acceptable, the "not investment advice" disclaimer wording, and the transcript/quote retention policy. Blocking for go-live, not for building.
6. **🔴 MANUAL — Approve the initial recipient list** and each analyst's coverage/angles (seed of `config/recipients.yaml`).
7. **🔴 MANUAL — Decide WhisperX host** (Phase 4): a GPU box / Databricks GPU job for the fallback. If the desk accepts *best-effort* coverage instead of 100%, skip WhisperX entirely and tell Claude to drop that branch.

**Store secrets** (🔴 MANUAL to create the store, 🤖 CLAUDE to wire reads):
- `PODSCAN_PAT`, `ANTHROPIC_API_KEY`, email-sender key, `HF_TOKEN` (Phase 4).
- Put them in a secret manager or, for the pilot, a local `.env` that is **git-ignored**. Never commit them.

### Phase 1 — Repo scaffolding & config (🤖 CLAUDE)

8. **🤖 CLAUDE — Scaffold the project.** Create `src/` (pipeline), `tests/`, `.env.example`, `requirements.txt` / `pyproject.toml`, and a `Makefile` / task runner. Add `.env` to `.gitignore`. Reuse the existing `config/podcasts.yaml` and `config/recipients.yaml` — do **not** invent new config formats.
9. **🤖 CLAUDE — Add `podscan_id` to each show in `config/podcasts.yaml`.** The current file keys shows by RSS; the Podscan job needs Podscan's `podcast_id`. Add a resolver step (Phase 2) that fills these in, and leave `rss` in place for the WhisperX fallback.
10. **🤖 CLAUDE — Define the claims schema** as a typed model (Pydantic/dataclass) matching the spec: `topic, tickers[], entities[], quote, timestamp, sentiment, angle_tags[]` (angle tags constrained to `config/recipients.yaml → angle_taxonomy`). This is the single interchange format — **no separate "evidence card" type.**

### Phase 2 — Podscan client, allow-list & smoke test (🤖 CLAUDE + one 🔴 verify)

11. **🤖 CLAUDE — Build a thin Podscan client** that sends the PAT as a `Bearer` token, parses the `quota` block from every response, and logs `daily_remaining`. Add a **rate limiter** capped well under 120 req/min. Add a startup **health check** that fails loudly if the PAT is invalid (§6).
12. **🔴 MANUAL — Verify tool behaviour in the MCP Console** (Podscan Dashboard → MCP Console). A human runs, with Claude watching the output:
    - exact parameter names for date filtering, `exclude_transcript`, `show_only_fully_processed`, and the search-highlight field;
    - whether `get-episode-transcript` returns full text or truncates at ~75K chars (and how to page it);
    - the shape of `get-podcast-episodes` / `get-recent-episodes` responses.
    Record findings; Claude codes to the **verified** names, not the assumed ones.
13. **🤖 CLAUDE — Configure the read-only tool allow-list** (§6.1) in the MCP client config. Assert in a test that no write/CRUD tool is reachable.
14. **🤖 CLAUDE — Resolve `podcast_id` for every enabled show** (via `search-podcasts` / `get-podcast`) and write them back into `config/podcasts.yaml`. **🔴 MANUAL:** a human eyeballs the matches (right show, right feed) before they're committed — name collisions are common.

### Phase 3 — Daily enumeration (🤖 CLAUDE)

15. **🤖 CLAUDE — Build the enumerator.** For each enabled `podcast_id`, call `get-podcast-episodes` / `get-recent-episodes` filtered to "since last watermark." Persist a per-show **watermark** (last processed episode id/date) in a small state file/table so runs are incremental.
16. **🤖 CLAUDE — Dedupe** candidates against the processed-id store; skip anything seen. This is the "never reprocess twice" guarantee — backed by real state, not hope.

### Phase 4 — Transcript acquisition with fallback (🤖 CLAUDE + 🔴 setup)

17. **🤖 CLAUDE — Podscan-first fetch.** For each new enabled episode, try `get-episode-transcript`. If present and complete, use it.
18. **🔴 MANUAL — Stand up WhisperX** (only if 100% coverage is required). Provision a GPU host/job. **Accept the `pyannote` model licence on Hugging Face and mint an `HF_TOKEN`** (diarization won't run without it — this is a per-model licence click a human must do). Install `whisperx` + CUDA deps.
19. **🤖 CLAUDE — Fallback branch.** If Podscan has no transcript for an enabled episode published today: download the audio from the show's `rss` enclosure and run WhisperX (word timestamps + diarization). Normalise Podscan and WhisperX outputs to **one** transcript format so the extractor doesn't care which produced it.
20. **🤖 CLAUDE — Optional escalation.** For a small number of high-priority misses, call `retranscribe-missing-episodes` (**≤5/day** — enforce the cap in code) to let Podscan process it for *next* run, while WhisperX covers *today*.

### Phase 5 — Extraction (🤖 CLAUDE)

21. **🤖 CLAUDE — Extract-once pass** (Claude Haiku). Transcript → list of typed **claims** (Phase 1 schema), each carrying a transcript **timestamp** for citation. Pre-fill entity/ticker/sentiment fields from Podscan's `get-episode-entities` / `get-company-episodes` (salience + ad-read-vs-editorial split) where available, rather than re-deriving them.
22. **🤖 CLAUDE — Persist claims** keyed by episode id (so tailoring for 40 recipients reads them without re-hitting Podscan or Claude-extract again).

### Phase 6 — Tailor & compose per recipient (🤖 CLAUDE)

23. **🤖 CLAUDE — Match/rank.** For each recipient in `config/recipients.yaml`, score every claim against `watchlist_tickers`, `coverage_sectors`, `angles` (embedding similarity + ticker/entity overlap + angle-tag match). Apply `avoid`, `max_items`, `depth`.
24. **🤖 CLAUDE — Compose** (Claude Sonnet) a digest **only from the selected, cited claims** — grounded, timestamped, with deep links. Never compose from raw transcript (keeps groundedness enforceable).
25. **🤖 CLAUDE — Render** HTML + plain-text, with the compliance footer and "not investment advice" disclaimer.

### Phase 7 — Email delivery (🤖 CLAUDE + 🔴 DNS)

26. **🔴 MANUAL — Choose & verify the sender.** SendGrid or Azure Communication Services (ACS) Email. Add **SPF, DKIM, DMARC** DNS records for the sending domain and verify — this needs DNS access and can take hours to propagate. (For a quick pilot only, the Cowork Gmail connector to a handful of pilot users is acceptable.)
27. **🤖 CLAUDE — Build the sender** against the chosen provider's API, reading the key from secrets.
28. **🤖 CLAUDE — Idempotent send ledger.** One row per `(recipient, digest_date)`; refuse to send a duplicate. This is what makes a re-run safe.

### Phase 8 — Scheduling (🤖 CLAUDE + 🔴 enable)

29. **🤖 CLAUDE — Wire the schedule.** A daily trigger (Claude Code on the web scheduled trigger, or a GitHub Actions cron) that runs enumerate → acquire → extract → tailor → send. Respect `poll_cron` in `config/podcasts.yaml`.
30. **🔴 MANUAL — Load secrets into the scheduler** (Actions secrets / trigger env) and enable the schedule. Do a **supervised first run** and read the logs before trusting it unattended.

### Phase 9 — Weekly discovery proposal (🤖 CLAUDE)

31. **🤖 CLAUDE — Separate weekly job.** Seed `get-similar-podcasts` from the top enabled shows; run `get-related-topics` / `search-topics` on the AI/semis/cloud ontology; use `get-trending-companies`. Output a **ranked list of candidate new shows** to a report/PR — **not** into the daily brief.
32. **🔴 MANUAL — Human enables shows.** An analyst/editor reviews the proposals and flips `enabled: true` on the ones worth adding. Discovery never auto-enables.

### Phase 10 — Evals & guardrails (🤖 CLAUDE + 🔴 golden set)

33. **🔴 MANUAL — Build a small golden set** (5–10 episodes with hand-checked expected claims + a couple of reference digests). Human judgment required.
34. **🤖 CLAUDE — Groundedness gate.** Every claim in a digest must cite a transcript timestamp; block/flag any that don't. Add a "no investment advice / no MNPI" output check.
35. **🤖 CLAUDE — Reconciliation check.** Compare the deterministic enumeration count against what was actually sent; **flag silent drops** (the spec's "no silent drops" rule). Alert on any enabled episode that got neither a Podscan nor a WhisperX transcript.

---

## 8. Manual-intervention summary (the whole 🔴 list in one place)

| When | Human action | Why it can't be automated |
|---|---|---|
| Before build | Provision Podscan account + Premium plan | Billing / identity |
| Before build | Mint Podscan **PAT** | UI-only secret, shown once |
| Before build | Anthropic API key | Billing / identity |
| Before build | Compliance sign-off (disclaimer, retention) | Judgment / accountability |
| Before build | Approve recipient list & angles | Business decision |
| Phase 2 | Verify tool params / transcript truncation in MCP Console | Confirms live schema before coding |
| Phase 2 | Eyeball `podcast_id` matches | Name collisions need a human check |
| Phase 4 | Stand up GPU + accept `pyannote` licence + `HF_TOKEN` | Infra + licence click |
| Phase 7 | Verify sender domain (SPF/DKIM/DMARC) | DNS access; slow propagation |
| Phase 8 | Load secrets into scheduler; supervise first run | Secret handling; trust gate |
| Phase 9 | Enable proposed new shows | Editorial decision |
| Phase 10 | Build golden eval set | Human ground truth |

---

## 9. Open items for the team

1. **Confirm the coverage promise.** If the desk can live with *best-effort same-day* instead of *100%*, we **delete the WhisperX branch** (Phase 4, steps 18–19) and the system loses its only piece of heavy infra. This is the single biggest simplification available — worth an explicit decision.
2. **Backfill.** The plan covers the daily steady state. Historical backfill (re-processing months of an enabled show) is a separate burst job; confirm depth defaults and whether it may borrow the daily request budget.
3. **Transcript truncation.** If `get-episode-transcript` can't return >75K chars in one call, long episodes need paging — confirm in Phase 2 so extraction never silently works on a half-transcript.
4. **Plan headroom.** Premium (2,000/day) is recommended; revisit if backfill + daily contend, or if the enabled universe grows well past ~150 shows.
