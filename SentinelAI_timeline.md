# 36-hour Timeline

> Built for a team of 2–4. Hours are elapsed, not wall-clock. Sleep is scheduled — don't skip it, you'll debug worse.

---

## Block 1 — Foundation (H0–H4)

| Hour | Focus | Person |
|---|---|---|
| 0:00 | Kickoff, read problem aloud, review architecture | Everyone |
| 0:30 | Repo setup, branch strategy, env files | Lead |
| 1:00 | Event schema + pydantic models (blocking dep) | Backend |
| 1:00 | Next.js scaffold + Tailwind + shadcn install | Frontend |
| 2:00 | Synthetic generators: network layer | ML/Data |
| 3:00 | Synthetic generators: endpoint + app layers | ML/Data |
| 3:30 | FastAPI skeleton + `/ingest` endpoint | Backend |
| 4:00 | **Checkpoint:** seed generator pushes events, API accepts them, DB stores them |

## Block 2 — Detection (H4–H12)

| Hour | Focus | Person |
|---|---|---|
| 4:00 | Rule detector: brute-force | Backend |
| 5:00 | Rule detector: lateral movement | Backend |
| 6:00 | Rule detector: data exfil | Backend |
| 7:00 | Rule detector: C2 beaconing | Backend |
| 5:00 | IsolationForest training harness | ML/Data |
| 7:00 | ML anomaly scorer integrated | ML/Data |
| 8:00 | LLM classifier + prompt v1 | Demo/Prompt |
| 9:00 | LLM caching layer + timeout/fallback | Demo/Prompt |
| 10:00 | Detection orchestrator wires rules + ML + LLM | Backend |
| 11:00 | Throughput test: 500 ev/s target | Backend |
| 11:30 | **Checkpoint:** all 4 categories detect in seeded data |
| 12:00 | **SLEEP — 4 hours minimum for at least half the team** |

## Block 3 — Correlation + MITRE (H12–H18)

| Hour | Focus | Person |
|---|---|---|
| 12:00 | Sliding-window correlator | Backend |
| 13:30 | Incident builder (from correlated detections) | Backend |
| 14:30 | MITRE mapping.json (hand-curated) | Demo/Prompt |
| 15:00 | MITRE enrichment in pipeline | Backend |
| 16:00 | Severity calculator + tests | Backend |
| 14:00 | Dashboard: incident list view | Frontend |
| 16:00 | Dashboard: incident detail view | Frontend |
| 17:30 | **Checkpoint:** cross-layer incident shows up in UI with MITRE tag |

## Block 4 — Playbooks + SOC report (H18–H22)

| Hour | Focus | Person |
|---|---|---|
| 18:00 | Playbook LLM prompt + JSON schema | Demo/Prompt |
| 19:00 | Static fallback templates | Backend |
| 20:00 | SOC report compiler | Backend |
| 20:00 | Dashboard: playbook panel + approve/reject | Frontend |
| 21:30 | **Checkpoint:** click incident → see playbook → approve → state persists |

## Block 5 — Dashboard polish + WebSockets (H22–H30)

| Hour | Focus | Person |
|---|---|---|
| 22:00 | WebSocket live feed backend | Backend |
| 23:00 | WebSocket client + real-time incident feed | Frontend |
| 24:00 | MITRE heatmap component | Frontend |
| 25:00 | **SLEEP — other half, 3–4h** |
| 25:00 | Dark theme, severity colors, micro-animations | Frontend |
| 27:00 | Analyst notes field + persistence | Full stack |
| 28:00 | Empty states, error states, loading skeletons | Frontend |
| 29:30 | **Checkpoint:** full dashboard feels production-ish |

## Block 6 — Simulation mode (H30–H33, stretch)

| Hour | Focus | Person |
|---|---|---|
| 30:00 | Scenario scripts (5 attacks) | Demo/Prompt |
| 31:00 | Replay engine + rate control | Backend |
| 32:00 | Coverage self-validation report | ML/Data |
| 32:30 | UI trigger button + live progress | Frontend |

## Block 7 — Demo prep (H33–H36)

| Hour | Focus | Person |
|---|---|---|
| 33:00 | Freeze code. Only bug fixes from here. | Everyone |
| 33:00 | Demo script written (5 min runtime) | Lead |
| 34:00 | Dry run #1 — full demo, time it | Everyone |
| 34:30 | Fix whatever broke | Everyone |
| 35:00 | Record backup video | Demo/Prompt |
| 35:30 | Dry run #2 — timed, no tweaks | Everyone |
| 35:45 | Slides final, screenshots final, GitHub README final |
| 36:00 | **Ship it.** |

---

## Risk register (things that will definitely go wrong)

| Risk | Mitigation |
|---|---|
| LLM API rate-limits during demo | Pre-run demo scenario once, cache all LLM responses for playback |
| Dashboard WebSocket disconnects | Auto-reconnect + last-N event buffer |
| Laptop dies during judging | Backup video + deploy to Vercel/Render as fallback |
| Synthetic data looks too obviously fake | Use real CICIDS sample for backup demo |
| ML model fits to seed = zero real signal | Split benign/malicious seeds, hold out validation set |
| Team member gets stuck for >30 min | Escalate to standup, pair-program or reassign |
