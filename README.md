# Podcast Synthesis Platform

Production-ready, per-recipient **email synthesis of technology & AI podcasts** for a
technology global equity investment team (up to 45 analysts & PMs). Podcasts can be
**dynamically added** and **backfilled**; shows without a transcript are **reliably
transcribed first**.

## What's here

| Path | Purpose |
|---|---|
| [`docs/product-specification.md`](docs/product-specification.md) | The full product specification — **start here** |
| [`config/podcasts.yaml`](config/podcasts.yaml) | Podcast source registry + declarative backfill jobs |
| [`config/recipients.yaml`](config/recipients.yaml) | Recipient profiles (coverage, angles, preferences) + angle taxonomy |

## Two streams of work

- **Stream A — Quick:** Claude Code + Cowork + small Python scripts. First email in **days**. Pilot / proof of value.
- **Stream B — Productionised:** Azure Databricks Lakehouse, **Mosaic AI Gateway**, **MLflow 3**, **LangChain/LangGraph**, and a **Configuration App**. Governed system of record with guardrails, LLM-judge evals, and OpenTelemetry.

Both streams share one data model and the same `config/` format, so the pilot graduates into the platform without rework.

## Question-by-question answers (in the spec)

- **All podcast providers & which have transcripts** → §3, §4
- **Free transcription (up to 90 min) & paid services** → §5
- **Backfill, selectable by podcast + days/episodes/date-range** → §6, `config/podcasts.yaml`
- **Per-recipient tailoring for 45 analysts/PMs** → §7, `config/recipients.yaml`
- **Logging, guardrails, evals, OpenTelemetry "gold standard," recorded** → §9
- **Azure Databricks architecture (AI Gateway, MLflow, LangChain, Config App)** → §10.3

## Status

Draft v1.0 — for investment-team review. See §14 for open decisions.
