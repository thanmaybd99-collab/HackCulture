# SentinelAI — Master Todo

> Rule: check items **only when proven working**, not when written. Never mark a task done without running it.

---

## Phase 0 — Idea submission (BEFORE offline hackathon)

- [ ] Lock the team (2–4 members) and roles (see README table)
- [ ] Export Langflow screenshots + a short flow description
- [ ] Write 1-slide problem framing + 1-slide "why now"
- [ ] Draw architecture diagram (use `docs/ARCHITECTURE.md` as source)
- [ ] List the 4 threat categories we will detect + MITRE techniques
- [ ] Define the demo scenarios (see `backend/data/samples/` specs below)
- [ ] Export PPT/PDF (9 slides max, judges read fast)
- [ ] Submit before deadline

---

## Phase 1 — Foundation (hours 0–4)

### 1.1 Repo + env
- [ ] `git init`, push to GitHub (private), add `.gitignore`, add `.env.example`
- [ ] Backend venv + `requirements.txt` pinned
- [ ] Frontend `npx create-next-app@latest frontend --typescript --tailwind --app`
- [ ] Pre-commit: `ruff` + `black` for Python, `prettier` for TS
- [ ] One-command dev runner in `scripts/dev.sh`

### 1.2 Unified event schema — **do this first, everything depends on it**
- [ ] Write `backend/app/core/schema.py` with `Event`, `Incident`, `Playbook` pydantic models
- [ ] Document every field in `docs/DATA_SCHEMA.md` with example JSON
- [ ] Add layer enum: `NETWORK | ENDPOINT | APPLICATION`
- [ ] Add category enum: `BRUTE_FORCE | LATERAL_MOVEMENT | DATA_EXFIL | C2_BEACON | BENIGN`
- [ ] Unit test: round-trip JSON ↔ Event for all three layer types

### 1.3 Synthetic data generators
- [ ] `generators/network.py` — NetFlow-like records (src/dst ip, port, proto, bytes, duration, tcp_flags)
- [ ] `generators/endpoint.py` — process exec records (pid, ppid, proc_name, user, cmdline, file_ops, reg_ops)
- [ ] `generators/application.py` — HTTP/API records (method, path, status, size, ua, geoip)
- [ ] Each generator has a `benign_stream()` and `malicious_stream(scenario)` function
- [ ] Demo seeder (`scripts/demo_seed.py`) spins up 2 attacks + 1 false positive simultaneously

---

## Phase 2 — Detection engine (hours 4–12)

### 2.1 Rule detectors (fast, interpretable, runs first)
- [ ] `detectors/rules/brute_force.py` — N failed auths from same src within W seconds
- [ ] `detectors/rules/lateral.py` — internal → internal connection to new host after auth success
- [ ] `detectors/rules/exfil.py` — outbound bytes > threshold OR unusual destination + size
- [ ] `detectors/rules/c2_beacon.py` — periodic low-volume connections to same ext IP (std dev of intervals check)
- [ ] Each rule returns `(matched: bool, confidence: float, reason: str, evidence: dict)`

### 2.2 ML anomaly layer
- [ ] IsolationForest on network feature vector (train on benign stream)
- [ ] Fit once at boot from `backend/data/samples/baseline.jsonl`, pickle the model
- [ ] Expose `detectors/ml/anomaly.py::score(event) -> float`
- [ ] Verify: benign events score near 0, injected anomalies score > 0.6

### 2.3 LLM classifier + explainer (the "wow" layer)
- [ ] `detectors/llm/classifier.py` — send event batch + rule/ML outputs, get category + confidence + plain-English reason
- [ ] System prompt lives in `detectors/llm/prompts.py` — tuned, versioned, not inline
- [ ] **Rate limit + cache** — hash event signature, cache results to avoid burning tokens in demo loop
- [ ] Fallback: if LLM fails/times out, use rule+ML verdict with generic reason

### 2.4 Detection orchestrator
- [ ] `detectors/engine.py::detect(event)` — runs rules → ML → (only if signal) LLM
- [ ] Returns `DetectionResult` with source labels (which rule matched, ML score, LLM verdict)
- [ ] Benchmark: 500 events/sec on laptop. If we can't hit it, parallelize with asyncio gather or batch.

---

## Phase 3 — Correlation + MITRE (hours 12–18)

### 3.1 Sliding window correlator
- [ ] `correlator/window.py` — async ring buffer indexed by `(src_ip, user, host)` over last 30s
- [ ] On new detection, check if same actor appears in another layer's buffer → boost confidence, create Incident
- [ ] Write 3 correlation cases as tests (network+endpoint, endpoint+app, all three)

### 3.2 MITRE ATT&CK mapping
- [ ] `mitre/mapping.json` — static table: category + sub-indicator → (tactic, technique_id, technique_name)
- [ ] `mitre/enrich.py::enrich(incident)` — attaches MITRE metadata to incident
- [ ] LLM fallback: if no static match, ask LLM to suggest technique (with low confidence flag)

### 3.3 Severity calculator
- [ ] `core/severity.py` — function of (confidence, cross-layer boost, MITRE tactic weight, asset criticality)
- [ ] Output: `LOW | MEDIUM | HIGH | CRITICAL`

---

## Phase 4 — Playbook + SOC report (hours 18–22)

- [ ] `playbook/generator.py` — LLM call with incident JSON + MITRE context → structured playbook
- [ ] Playbook schema: `{ immediate_actions[], investigation_steps[], containment[], eradication[], recovery[], iocs[] }`
- [ ] Static fallback playbooks per category in `playbook/templates/`
- [ ] SOC report compiler: bundles incident + evidence + playbook + analyst fields (approve/reject/notes)

---

## Phase 5 — Dashboard (hours 22–30)

- [ ] Layout: top bar (alert count, severity breakdown), left panel (live incident feed), center (incident detail + playbook), right (MITRE heatmap)
- [ ] WebSocket live feed of incidents from backend
- [ ] MITRE ATT&CK matrix heatmap (tactic columns × technique rows, colored by hit count)
- [ ] Incident detail modal: timeline, evidence, LLM reasoning, playbook with approve/reject buttons
- [ ] "Analyst notes" free-text field persisted back to backend
- [ ] Dark theme by default (SOC aesthetic), high-contrast severity colors

---

## Phase 6 — Simulation mode (hours 30–33, stretch)

- [ ] `simulation/scenarios/` — 5 attack scripts (brute-force, lateral, exfil, C2, mixed-APT)
- [ ] Replay engine streams events at configurable rate
- [ ] Self-validation: after scenario ends, check if expected incidents were detected → print coverage report
- [ ] UI button: "Run simulation" → shows live coverage percentage

---

## Phase 7 — Demo prep + polish (hours 33–36)

- [ ] Seed dataset locked (2 active attacks + 1 false positive, reproducible seed)
- [ ] Demo script written (5-min walkthrough), timed twice
- [ ] Record a 90-sec backup video in case demo laptop fails
- [ ] Slide deck updated with real screenshots
- [ ] README polished, GitHub README has screenshots + GIF
- [ ] Final dry run with full team

---

## Definition of done (per phase)

A phase is done when:
1. All its tasks are checked
2. `pytest` passes (where applicable)
3. The demo seed still works end-to-end
4. A staff engineer would approve the code (per our principles)

---

## Review section (fill in after each phase)

### Phase 1 review
_pending_

### Phase 2 review
_pending_

### Phase 3 review
_pending_

### Phase 4 review
_pending_

### Phase 5 review
_pending_

### Phase 6 review
_pending_

### Phase 7 review
_pending_
