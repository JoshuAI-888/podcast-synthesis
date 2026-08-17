# Off-the-Shelf Solutions — Build vs. Buy Evaluation

**Companion to** [`product-specification.md`](product-specification.md) and [`architecture-and-build-options.md`](architecture-and-build-options.md)
**Purpose:** survey what you could *buy* instead of build, map each to our requirements, and recommend where to buy vs. build.
**Date:** 2026-08-17 · **Status:** Draft v1.0

---

## 0. The headline

**No single off-the-shelf product does the whole chain** — *ingest arbitrary tech/AI podcasts → reliably transcribe the ones without transcripts → backfill by podcast/days/episodes → synthesise per-recipient by investment angle for 45 analysts → deliver governed, evaluated email inside your Azure tenancy.*

But several products **cover individual layers very well** and can meaningfully cut build effort. The right answer is a **hybrid**: *buy the commodity layers (podcast discovery, transcription, mention search), build the differentiated layer (per-recipient investment synthesis + governance/evals)*. This section shows why, product by product.

The requirement set we score against:

| # | Requirement |
|---|---|
| R1 | Add **any** podcast dynamically (open RSS, not a fixed catalogue) |
| R2 | **Backfill** selectable by podcast + days/episodes/date-range |
| R3 | **Reliable transcription** (up to 90 min, diarized) of shows lacking transcripts |
| R4 | **Per-recipient** synthesis by investment angle (up to 45 analysts/PMs) |
| R5 | **Governed email** delivery, production-grade |
| R6 | **Logging, guardrails, evals, OpenTelemetry**, recorded |
| R7 | **In-tenancy / Azure-Databricks** data residency & compliance |

---

## 1. The landscape, by category

### 1.1 Buy-side research platforms (adjacent — you may already own one)
| Product | What it does | Fit for *this* job |
|---|---|---|
| **AlphaSense (+ Tegus)** | Market-intelligence platform: 200k+ expert-call transcripts, earnings, filings, broker research, news; **custom AI agents** (Channel Checks, Earnings Analysis, scheduled delivery); MNPI-mitigation controls | **Strong complement, not a replacement.** It indexes *its own* corpus (expert calls, earnings, news) — you **cannot point it at an arbitrary list of tech/AI podcasts** and get per-analyst, angle-tailored digests your way. Best used *alongside* our platform as another source. |
| **Bloomberg / Sentieo (AlphaSense)** | Terminal news/research, some audio/event transcripts | Same shape — curated corpus, not "your chosen podcasts, your angles." |

> If the team already licenses AlphaSense, its **scheduled AI agents** are the closest thing to "buy," but only over AlphaSense content. Podcasts of your choosing remain a build/assemble problem.

### 1.2 Podcast monitoring / intelligence APIs (the best "buy" for R1–R3)
| Product | What it does | Notes |
|---|---|---|
| **Podscan.fm** ⭐ | Transcribes **all** podcasts continuously; keyword/semantic **alerts** via email/webhook/**Slack/Teams**; full **REST API** over millions of episodes | Can serve as a ready-made **ingest + transcribe + search/backfill substrate**, removing the need to run WhisperX and a crawler. External SaaS (data leaves tenancy). |
| **CastFox** | Search **114M+ episode transcripts** across 5M+ shows; mention monitoring | Good for backfill/mention search breadth. |
| **Listen Notes** (API + Alerts) | Podcast search/metadata API; keyword alerts | Great for **discovery + deep backfill listings**; lighter on transcription. |
| **Rephonic / Podchaser Alerts / Syften** | Podcast databases + mention alerts | Monitoring-oriented; less of a full data pipe. |

**Why this category matters:** Podscan/Listen Notes can collapse our R1–R3 (discovery + transcription + backfill search) into an API call. That is the single biggest "buy" lever available.

### 1.3 Enterprise market-intelligence + auto-newsletter (the best "buy" for R4–R5, text-first)
| Product | What it does | Fit |
|---|---|---|
| **Feedly Market Intelligence** | "AI Feeds" that think like an analyst across millions of sources; **automated, audience-specific newsletters** with AI overviews; Slack/Teams + API | Excellent at the **tailored-newsletter** shape and stakeholder segmentation — but it's **text-source-first**; podcast *audio* is only via custom RSS with **no transcription of the audio**. Enterprise pricing (~$1.6k–2.4k/mo+). |

### 1.4 Knowledge-synthesis tools (research aids, not pipelines)
| Product | What it does | Fit |
|---|---|---|
| **Google NotebookLM (Enterprise / Agentspace)** | Grounded Q&A + **audio overviews** with citations over documents you add | Superb for *ad-hoc* research over a set of episodes; **not** a scheduled, per-recipient email pipeline; and it's **Google Cloud**, not your Azure estate. |
| **Microsoft Copilot / M365** | Summarisation across your tenant | In-Azure-adjacent, but not podcast-ingest shaped. |

### 1.5 Creator "repurposing" tools (wrong buyer, but capable)
| Product | What it does | Fit |
|---|---|---|
| **Castmagic · Swell AI · Podsqueeze · Capsho** | Upload an episode → show notes, blog, social, **newsletter draft**; ~$1–2/episode | Built for **podcast creators publishing one show**, not consumers synthesising *many* shows per *many* recipients. No investment angles, no governance, no backfill orchestration. Not fit for the desk. |

### 1.6 Personal podcast summarisers (out of scope)
**Snipd · Podwise · BibiGPT** — consumer apps for listeners (highlights, chat-with-episode). Great personally; irrelevant as an enterprise pipeline.

### 1.7 Transcription-as-a-service (a component, already in the spec)
**Deepgram · AssemblyAI · Azure AI Speech · Speechmatics · OpenAI** — buy R3 as a managed API instead of self-hosting WhisperX. Covered in `product-specification.md` §5.

### 1.8 Azure / Databricks assembly (low-code "buy-ish")
**Azure AI Speech + Azure OpenAI + Logic Apps / Power Automate**, or **Databricks solution accelerators** — not a product, but a faster assembly of first-party parts. Keeps everything in-tenancy (good for R7) at the cost of building the glue yourself.

---

## 2. Coverage matrix — nothing covers it all

✅ strong · ◐ partial · ✗ no / not its job

| Solution | R1 add any | R2 backfill | R3 transcribe | R4 per-recipient angle | R5 email | R6 evals/OTel | R7 in-tenancy |
|---|---|---|---|---|---|---|---|
| **AlphaSense (+Tegus)** | ✗ (own corpus) | ◐ | ✅ (own) | ◐ (agents) | ✅ | ✗ (opaque) | ✗ |
| **Podscan.fm** | ✅ | ✅ | ✅ | ✗ | ◐ (alerts) | ✗ | ✗ |
| **Listen Notes / CastFox** | ✅ | ✅ | ◐ | ✗ | ◐ | ✗ | ✗ |
| **Feedly MI** | ◐ (RSS text) | ◐ | ✗ (no audio ASR) | ◐ (segments) | ✅ | ✗ | ✗ |
| **NotebookLM Enterprise** | ◐ (manual add) | ✗ | ✅ | ✗ | ✗ | ✗ | ✗ (GCP) |
| **Creator tools (Castmagic…)** | ◐ | ✗ | ✅ | ✗ | ◐ (draft) | ✗ | ✗ |
| **Managed STT (Deepgram/Azure…)** | n/a | n/a | ✅ | ✗ | ✗ | ✗ | ◐ (Azure ✅) |
| **Our build (Stream B)** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

The gap every third-party leaves is the **same one**: R4 (your investment angles, per analyst) + R6 (your governance/evals) + R7 (your tenancy). Those three are exactly the **differentiated, compliance-sensitive** parts — the things worth building.

---

## 3. Recommendation — a hybrid, and how it changes the two streams

### 3.1 Principle: buy the commodity, build the differentiator
- **Commodity (buy or use managed):** podcast discovery, audio transcription, mention search/backfill breadth. These are undifferentiated and well-served — Podscan/Listen Notes or managed STT.
- **Differentiated (build):** per-recipient investment-angle synthesis, compliance guardrails, evals/OTel, and delivery — no vendor does this *your* way, and it's where the value (and the MNPI/no-advice risk) sits.

### 3.2 Concrete "buy" levers to fold into the plan
| Lever | Replaces build effort | Cost of buying |
|---|---|---|
| **Podscan.fm API** for ingest + transcription + backfill search | Running WhisperX + a crawler + a mention index | External data residency (R7), per-use fee; keep WhisperX as the in-tenancy fallback |
| **Listen Notes** for deep backfill listings | The "feed truncated" problem in R2 | Modest API cost |
| **Managed STT (Azure AI Speech)** instead of self-hosted | GPU ops for R3 | ~$1/hr, but stays in Azure (R7 ✅) |
| **AlphaSense scheduled agents** (if already licensed) | A parallel intelligence stream over expert calls/earnings | Existing spend; complements, doesn't replace |

### 3.3 How this reshapes the streams
- **Stream A (Quick) gets faster still:** wire **Podscan/Listen Notes** for ingest+transcripts and let Claude do the per-recipient synthesis. You could have angle-tailored emails from real transcripts in **days with almost no transcription code** — at the cost of sending data to a SaaS during the pilot (acceptable for a time-boxed proof).
- **Stream B (Production) stays build-led on R4/R6/R7, with a make-or-buy switch on R3:** default **WhisperX in-tenancy** for residency; keep a **config flag** to route a show to **Podscan/managed STT** when convenient. The synthesis, guardrails, evals and delivery remain ours on Databricks — that's the moat and the compliance boundary.

### 3.4 What to *not* buy
- **Creator repurposing tools** (Castmagic/Swell/Podsqueeze) — wrong buyer, single-show, no angles/governance.
- **A platform as the "brain"** (Feedly/NotebookLM as the synthesis engine) — you'd inherit their model, their tenancy, and lose per-analyst angle control and your eval gates.

### 3.5 One-line verdict
> **Buy the pipes (discovery/transcription/backfill), build the brain (per-recipient synthesis + governance).** Evaluate **Podscan.fm** as the fastest accelerator for R1–R3, keep **Azure AI Speech / WhisperX** as the in-tenancy transcription path, treat **AlphaSense** as a complementary source rather than a substitute, and keep synthesis, evals and delivery on Databricks where residency and differentiation demand it.

---

## 4. Suggested evaluation actions (before committing)
1. **Trial Podscan.fm API**: transcript quality on 5 target shows, backfill depth, latency after publish, pricing at your volume, and data-handling/DPA terms (R7).
2. **Check existing AlphaSense entitlement** and whether scheduled agents already cover part of the need (avoid duplicate spend).
3. **Benchmark managed STT vs. WhisperX** on 3 hard-audio episodes (accents/noise) → sets the R3 default per show.
4. **Confirm data-residency posture** with compliance for any external SaaS in the loop; decide pilot-only vs. production use.

---

## Appendix — sources
- Podscan.fm (monitoring + REST API) — https://podscan.fm/ · https://podscan.fm/rest-api
- CastFox / Rephonic / Podchaser / Syften (podcast monitoring) — https://www.castfox.net/use-cases/podcast-monitoring · https://rephonic.com/podcast-monitoring · https://www.podchaser.com/alerts · https://syften.com/podcast
- AlphaSense + Tegus (expert transcripts, AI agents) — https://www.alpha-sense.com/press/alphasense-scales-workflow-automation-in-financial-firms-and-enterprises/ · https://help.alpha-sense.com/hc/en-us/articles/43785894151699-Introduction-to-Tegus-Expert-Transcript-Library
- Feedly Market Intelligence (AI feeds + newsletters) — https://www.gartner.com/reviews/product/feedly-market-intelligence
- Creator repurposing (Castmagic/Swell/Podsqueeze) & summarisers — https://www.fastcompany.com/91498696/the-best-ai-podcast-summary-tools-to-save-time-and-find-highlights-in-2026 · https://bibigpt.co/en/blog/posts/best-ai-podcast-summarizer-tools-2026
