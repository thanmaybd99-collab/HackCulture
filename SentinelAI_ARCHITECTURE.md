# SentinelAI — Technical Architecture

## Design principles

1. **Layered detection, not monolithic model.** Rules catch known patterns cheap. ML catches anomalies. LLM explains and classifies edge cases. Each layer is independently testable.
2. **Events are first-class.** Everything in the system is an `Event` (raw log) or an `Incident` (correlated group of events). No other data shapes.
3. **Correlation is where confidence comes from.** A single detection is a lead, not a verdict. Cross-layer correlation is the forcing function for severity escalation.
4. **Explainability is not optional.** Every alert carries its provenance (which rule/model/LLM fired, with what evidence).
5. **Deterministic demo, probabilistic production.** The seeded scenarios are reproducible byte-for-byte. The pipeline itself is async and probabilistic.

---

## Component map

### 1. Ingestion (`backend/app/api/ingest.py`)

- `POST /ingest` — single event or batch
- `WS /stream` — push source (for generators)
- Schema-validates against `Event` pydantic model
- Routes into the async event bus (`core/bus.py`) — a per-worker `asyncio.Queue`

### 2. Normalization (`backend/app/core/normalize.py`)

- Each layer has its own raw-to-Event adapter
  - `normalize_network(raw) -> Event`
  - `normalize_endpoint(raw) -> Event`
  - `normalize_app(raw) -> Event`
- Fills common fields: `ts`, `actor`, `target`, `layer`, `raw_payload`
- Deterministic: same input → same output (important for caching)

### 3. Detection engine (`backend/app/detectors/`)

```
detectors/
├── engine.py           # orchestrator — run order: rules → ML → LLM
├── rules/
│   ├── brute_force.py
│   ├── lateral.py
│   ├── exfil.py
│   └── c2_beacon.py
├── ml/
│   ├── anomaly.py      # IsolationForest wrapper
│   └── features.py     # feature extraction per layer
└── llm/
    ├── classifier.py   # takes event + prior signals → category + reason
    ├── prompts.py      # versioned system prompts
    └── cache.py        # signature-hash LRU
```

#### Detection flow per event

```
event
  │
  ▼
┌─────────────┐
│ Rule engine │ ──► matched? yes→ signal, no → continue
└─────────────┘
  │
  ▼
┌─────────────┐
│ ML anomaly  │ ──► score > 0.6? yes→ signal, else → probably benign
└─────────────┘
  │  (only if rules OR ML flagged)
  ▼
┌─────────────┐
│ LLM verify  │ ──► confirms + reason + MITRE hint
└─────────────┘
  │
  ▼
DetectionResult
```

### 4. Correlator (`backend/app/correlator/`)

- Ring buffer keyed by `(actor)` where actor = src_ip / user / host depending on layer
- Window: 30 seconds default (configurable)
- On new `DetectionResult`:
  1. Add to buffer
  2. Look up same actor across other layers' buffers
  3. If ≥2 layers within window → create `Incident`, boost confidence
  4. Otherwise → single-layer `Alert` (lower priority)

Correlation rules table:

| Pair | Implication |
|---|---|
| network brute-force + endpoint new-login-success | successful compromise |
| endpoint process-spawn + network internal-east-west | lateral movement confirmed |
| endpoint suspicious-process + network large-outbound | exfil in progress |
| network periodic-beacon + endpoint persistent-process | C2 channel established |

### 5. MITRE enrichment (`backend/app/mitre/`)

- `mapping.json` — hand-curated table, format:
  ```json
  {
    "brute_force": {
      "tactic": "TA0006 Credential Access",
      "technique_id": "T1110",
      "technique_name": "Brute Force"
    }
  }
  ```
- For sub-indicators (e.g. "distributed brute force" → `T1110.003`), nested keys
- LLM fallback for uncategorized: prompts for tactic/technique suggestion with `needs_review: true`

### 6. Severity (`backend/app/core/severity.py`)

```
severity = f(
  base_confidence,        # from detection
  cross_layer_boost,      # +0.2 per additional layer
  mitre_tactic_weight,    # initial-access < exfil < impact
  asset_criticality       # from static asset tag table
)

→ LOW   (0.0–0.3)
→ MED   (0.3–0.6)
→ HIGH  (0.6–0.85)
→ CRIT  (0.85–1.0)
```

### 7. Playbook generator (`backend/app/playbook/generator.py`)

- Input: `Incident` (with MITRE context)
- LLM call with structured output schema:
  ```
  {
    "immediate_actions": ["..."],
    "investigation_steps": ["..."],
    "containment": ["..."],
    "eradication": ["..."],
    "recovery": ["..."],
    "iocs": [{"type": "ip", "value": "..."}],
    "references": ["MITRE T1110 mitigation"]
  }
  ```
- Static templates in `playbook/templates/{category}.json` as fallback
- All playbooks versioned in DB for audit trail

### 8. Dashboard (`frontend/`)

Pages:
- `/` — live SOC overview (incident feed, severity breakdown, MITRE heatmap)
- `/incidents/[id]` — incident detail + playbook + analyst actions
- `/simulation` — scenario runner (if simulation mode lands)

Key components:
- `<IncidentFeed />` — WebSocket-driven, auto-scrolling
- `<MitreHeatmap />` — tactics × techniques grid
- `<PlaybookPanel />` — collapsible steps with checkboxes + approve/reject
- `<EvidenceTimeline />` — per-incident event timeline

### 9. Simulation engine (`simulation/`)

- Scenarios are JSON scripts:
  ```json
  {
    "name": "apt_campaign",
    "duration_sec": 180,
    "actors": [...],
    "phases": [
      {"t": 0, "layer": "network", "type": "brute_force", ...},
      {"t": 45, "layer": "endpoint", "type": "persist", ...},
      ...
    ],
    "expected_incidents": ["brute_force", "lateral_movement", "exfil"]
  }
  ```
- Player emits events to `/ingest` at scheduled times
- After run: query backend for incidents, compare to expected → coverage %

---

## Data flow (happy path, end-to-end)

```
generator → POST /ingest → bus.Queue
                              │
                              ▼
                         normalize → Event
                              │
                              ▼
                       detection engine
                              │
                              ▼
                       DetectionResult
                              │
                              ▼
                         correlator
                              │
                              ▼
                         Incident (+ MITRE)
                              │
                              ▼
                       playbook generator
                              │
                              ▼
                   persist to DB + WS broadcast
                              │
                              ▼
                        Dashboard live
                              │
                              ▼
                    Analyst approves/rejects
                              │
                              ▼
                     final state persisted
```

---

## Non-goals (explicitly out of scope)

- Real network packet capture (we simulate)
- Real endpoint agent (we simulate)
- Multi-tenant auth/RBAC
- Persistent queue (Redis/Kafka) — in-memory asyncio is enough for demo
- Training ML models from scratch on large datasets — we use small, fitted-at-boot models
- Integrating with real SIEMs (Splunk, Elastic) — we are the SIEM
