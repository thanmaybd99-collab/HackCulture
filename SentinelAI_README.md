# SentinelAI — AI-Driven Threat Detection & Simulation Engine

> Hack Malenadu '26 | Cybersecurity Track | Team Size: 2–4 | Duration: 36h

---

## What we're building (in one paragraph)

SentinelAI ingests logs from three layers (network, endpoint, application), normalizes them into a unified event schema, runs a hybrid detection engine (rules + ML anomaly detection + LLM classification), correlates events across layers in a sliding time window, classifies threats against MITRE ATT&CK with a confidence/severity score, and generates a dynamic response playbook per incident. A live SOC dashboard shows incidents in real time; a simulation mode lets defenders replay attack scenarios to validate detection coverage.

## The pitch (30-second version for judges)

> "Traditional SIEMs drown SOC analysts in alerts without context. SentinelAI flips the model: every alert arrives with a plain-English explanation of *why* it was flagged, a MITRE ATT&CK mapping of *what* the attacker is doing, and a dynamically generated playbook of *what to do next* — ready for analyst approval in seconds, not hours."

## Architecture at a glance

```
┌─────────────────────────────────────────────────────────────────┐
│                      DATA GENERATION LAYER                      │
│  synthetic: network flows + process execs + HTTP API logs       │
│  (also supports CICIDS / UNSW-NB15 replay)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │  500+ events/sec
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│               L1 — INGESTION & NORMALIZATION                    │
│  FastAPI /ingest  →  schema validator  →  unified Event model   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                 L2 — DETECTION ENGINE (hybrid)                  │
│  ┌──────────┐   ┌──────────────┐   ┌────────────────────────┐   │
│  │  Rules   │ + │ ML Anomaly   │ + │ LLM Classifier +       │   │
│  │  engine  │   │ (IsolForest) │   │ Explainer (Claude/GPT) │   │
│  └──────────┘   └──────────────┘   └────────────────────────┘   │
│  → 4 threat classes: brute-force, lateral, exfil, C2 beacon     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│             L3 — CORRELATION + MITRE MAPPING                    │
│  sliding window (5–30s)  →  cross-layer match  →  incident      │
│  MITRE ATT&CK tactic/technique mapping + severity calculator    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│               L4 — PLAYBOOK GENERATOR (LLM)                     │
│  context-aware response steps  →  SOC report  →  audit trail    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   L5 — SOC DASHBOARD (Next.js)                  │
│  live incident feed  │  MITRE heatmap  │  playbook approval UI  │
└─────────────────────────────────────────────────────────────────┘

            ┌──────────── SIMULATION ENGINE (bonus) ───────────┐
            │  replays attack scenarios → self-validates flow  │
            └──────────────────────────────────────────────────┘
```

## Repo layout

```
sentinelai/
├── README.md                     ← you are here
├── tasks/
│   ├── todo.md                   ← the plan (check items as we go)
│   ├── lessons.md                ← rules we learn the hard way
│   └── timeline.md               ← 36-hour hour-by-hour schedule
├── docs/
│   ├── ARCHITECTURE.md           ← deeper technical breakdown
│   ├── TECH_STACK.md             ← why we picked what we picked
│   ├── DATA_SCHEMA.md            ← unified event schema spec
│   └── MITRE_MAPPING.md          ← threat → technique table
├── backend/                      ← FastAPI service
│   ├── app/
│   │   ├── api/                  ← HTTP routes (ingest, incidents, ws)
│   │   ├── core/                 ← config, logging, event schema
│   │   ├── detectors/            ← rule + ML + LLM detectors
│   │   ├── correlator/           ← sliding-window correlation
│   │   ├── playbook/             ← LLM playbook generation
│   │   ├── mitre/                ← ATT&CK mapping table + helpers
│   │   ├── models/               ← pydantic + DB models
│   │   └── utils/
│   ├── data/
│   │   ├── generators/           ← synthetic log producers
│   │   └── samples/              ← seeded demo scenarios
│   ├── tests/
│   └── requirements.txt
├── frontend/                     ← Next.js SOC dashboard
├── langflow/                     ← the idea-phase flow exports
├── simulation/                   ← attack scenario player
└── scripts/                      ← dev helpers, seeders, demo runners
```

## Tech stack (summary — see `docs/TECH_STACK.md` for reasoning)

| Layer | Choice |
|---|---|
| Backend API | Python 3.11 + FastAPI + Uvicorn |
| Event stream | In-memory asyncio queue (Redis optional) |
| Storage | DuckDB (fast analytics) + SQLite (state) |
| ML | scikit-learn (IsolationForest, DBSCAN) |
| LLM | OpenRouter (Claude Sonnet / GPT-4o / Gemini) |
| Frontend | Next.js 14 + Tailwind + Recharts + shadcn/ui |
| Realtime | WebSocket (FastAPI `websockets`) |
| Langflow | Idea-phase demo flow (already built ✅) |

## Quick start (will be filled in as we build)

```bash
# backend
cd backend && python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload

# frontend
cd frontend && npm install && npm run dev

# demo seed (runs a brute-force + C2-beacon + false-positive scenario)
python scripts/demo_seed.py
```

## Team

_fill in before idea-phase submission_

| Role | Name | Focus |
|---|---|---|
| Lead / Backend | | detection engine + correlation |
| ML / Data | | synthetic data + anomaly model |
| Frontend / UX | | SOC dashboard + playbook UI |
| Demo / Prompt | | LLM prompts + simulation + pitch |
