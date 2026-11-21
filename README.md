# LineReady: Line Engineer Agent

An AI assistant for aircraft line engineers (AI Hackathon, Use Case 4). Before the night shift starts it has
already prepared every defect on the engineer's aircraft: the symptom, the aircraft and fleet history, the exact
TSM/AMM task at the controlled revision, the parts and tools in stores, and the deadlines. Every line links to
its source record. **The engineer decides and signs; LineReady never signs, defers or releases anything.**

> **The application is the tablet UI at http://localhost:3000** (redirects to `/tonight`).
> Port 8000 is the backend API that the UI calls; you do not need to open it.

---

## Contents

1. [Ports](#1-ports)
2. [What is in git, and what is not](#2-what-is-in-git-and-what-is-not)
3. [Prerequisites](#3-prerequisites)
4. [Run it with a knowledge-base file (fastest)](#4-run-it-with-a-knowledge-base-file-fastest)
5. [Create the knowledge base from the data pack](#5-create-the-knowledge-base-from-the-data-pack)
6. [Switch on Claude](#6-switch-on-claude)
7. [Using the app](#7-using-the-app)
8. [How it works](#8-how-it-works)
9. [Accuracy and tests](#9-accuracy-and-tests)
10. [Deploy to Kubernetes (2 GB disk)](#10-deploy-to-kubernetes-2-gb-disk)
11. [Configuration](#11-configuration)
12. [Project layout](#12-project-layout)
13. [Troubleshooting](#13-troubleshooting)
14. [Known limits](#14-known-limits)

---

## 1. Ports

| URL | Service | Use |
|---|---|---|
| **http://localhost:3000** | `web` (Next.js) | **The application.** Open this. |
| http://localhost:8000/api/... | `app` (FastAPI) | Backend API used by the UI (through its `/backend` proxy) |
| http://localhost:8000/docs | `app` | API documentation (Swagger), for developers |
| http://localhost:8000/ | `app` | Old single-page debug UI from Round 1. Not the product. |
| localhost:5433 | `db` (Postgres + pgvector) | Database, for developers |

All ports bind to `127.0.0.1` only. In Kubernetes only the UI is published through the Ingress (Service port
80 → container 3000); the API (8000) and the database stay inside the cluster.

## 2. What is in git, and what is not

| Item | In git | Where it comes from |
|---|---|---|
| Code, Dockerfiles, Kubernetes manifests, scripts | yes | this repo |
| Data pack (`01 Core`, `02 Supplementary`, 1.9 GB) | **no** | the hackathon organisers. Proprietary (Airbus manuals, AMOS data) |
| Knowledge-base file `kb/lineready-kb-<date>.dump` (185 MB) | **no** | built from the data pack with `scripts/build_kb.sh` ([section 5](#5-create-the-knowledge-base-from-the-data-pack)) |
| `.env` with the Claude API key | **no** | you create it ([section 6](#6-switch-on-claude)) |

A `git clone` alone does not run the app: you also need **either** the knowledge-base file **or** the data pack.
Share the `.dump` file only through a secure channel: it contains Airbus manual text and AMOS records.

## 3. Prerequisites

* Docker with Docker Compose v2 (no Python or Node needed on the host).
* 8 GB RAM recommended for building the knowledge base; ~5 GB free disk for images and the database.
* Internet access the first time you build the images (Python/Node packages and the embedding model).
* Optional: a Claude API key.

## 4. Run it with a knowledge-base file (fastest)

Use this when someone gives you a ready `lineready-kb-<date>.dump`. About 2 minutes plus the image build.

```bash
git clone https://github.com/TheshikaSamaraweera/lineready.git
cd lineready
mkdir -p kb && cp /path/to/lineready-kb-20261122.dump* kb/      # the .dump and its .sha256

docker compose up -d db                                          # 1. database
./scripts/restore_kb.sh kb/lineready-kb-20261122.dump            # 2. load the knowledge base (~40 s)
docker compose up -d --build                                     # 3. API + UI
```

Open **http://localhost:3000**. Each engineer's first brief is built when first opened (a few seconds).

## 5. Create the knowledge base from the data pack

Do this once on a laptop or build machine, and again whenever the data pack changes. The result is one file you
can give to teammates and stream into the server.

### 5.1 Put the data pack next to the repo

```
hackthon/
├── 01 Core/                 ← data pack
├── 02 Supplementary/        ← data pack
└── lineready/               ← this repo
```

`docker-compose.yml` mounts `../01 Core` and `../02 Supplementary` read-only into the build container.

### 5.2 Run the build

```bash
cd lineready
./scripts/build_kb.sh
```

It runs, in order (all inside Docker; nothing leaves the machine):

| Step | Command it runs | Time | Result |
|---|---|---|---|
| Schema | `cli init` | seconds | tables, indexes, pgvector |
| AMOS | `cli ingest wo stock tools` | ~30 s | 209,421 work orders, 62,295 stock lots, 11,989 tools |
| AVIATAR | `cli ingest faults pha` | ~10 s | 375,894 fault messages, 199 predictive cases |
| TSM | `cli ingest tsm` | ~7 min | 9,125 tasks, 55,680 fault-index rows, revision TSN 86 |
| Fleet | `cli ingest fleet` | seconds | 92 aircraft: registration → MSN → effectivity code (from the TSM) |
| AMM / IPC | `cli ingest amm ipc` | ~2 min incl. extraction | 11,594 AMM tasks, 332,547 IPC items |
| Case memory | `cli ingest cases` | seconds | 4,887 past defects with the TSM task engineers used |
| Embeddings | `cli ingest embed qembed` | ~10-20 min on CPU | 81,752 defect + 20,719 task-title vectors; 4,136 precomputed query vectors |
| Demo shift | `cli seed-shift` | seconds | synthetic SHJ night: 3 engineers, 10 aircraft |
| Trim | SQL | seconds | drops scheduled task cards (not used at runtime) |
| Export | `pg_dump` | ~1 min | **`kb/lineready-kb-<date>.dump`** (~185 MB) + `.sha256` |

Re-export without rebuilding: `./scripts/build_kb.sh --export`.

### 5.3 Run the steps by hand (optional)

```bash
docker compose up -d db app
docker compose exec app python -m lineready.interfaces.cli init
docker compose exec app python -m lineready.interfaces.cli ingest wo stock tools faults pha tsm
docker compose exec app python -m lineready.interfaces.cli ingest fleet amm ipc cases
docker compose exec app python -m lineready.interfaces.cli ingest embed qembed
docker compose exec app python -m lineready.interfaces.cli seed-shift
docker compose up -d web
```

`ingest` with no names runs every loader in order (the same as all the lines above); naming loaders runs only those.

### 5.4 Check the knowledge base

```bash
curl localhost:8000/api/health      # row counts and TSM revision
curl localhost:8000/api/readyz      # {"ok":true,"tsm":"TSN 86"}
```

## 6. Switch on Claude

Without a key everything works in **template mode**: summaries are written by rules; Ask and the Coach feedback
are unavailable. To use Claude:

```bash
cp .env.example .env
# edit .env:  ANTHROPIC_API_KEY=sk-ant-...      (one line, KEY=value form)
docker compose up -d --force-recreate app
```

Then press **Rebuild brief** in the UI. Token controls:

* Only the top `LLM_NARRATE_TOP` items per brief (default 5) get an AI summary; the rest use templates.
* Every Claude response is cached in the database: rebuilding with unchanged data costs nothing.
* Measured: one brief ≈ 10 calls ≈ 4k output tokens; one Ask ≈ 1.2k output tokens; Coach and Handover 1 call each.

## 7. Using the app

Open http://localhost:3000. Choose the engineer and role under **Me** (bottom-left of the rail).

| Screen | Who | What it shows |
|---|---|---|
| **Tonight** | engineer, lead | The shift brief: aircraft on a ground timeline, "Next up", items grouped by aircraft, readiness gaps, last handover |
| **Dossier** | engineer | One defect: summary, timeline, history (did past fixes hold?), TSM/AMM references, parts and tools; Accept / Edit / Not relevant (logged) |
| **Aircraft** | engineer, lead | Pack status per aircraft; packs are built 90 min before landing. The **station clock** chips replay the night (21:00 → 03:00) |
| **Pack** | engineer | Everything for one turnaround: items, things to clear while on the ground, parts, tools, manual pages, last 7 days |
| **Manuals** | engineer, trainee | TSM and AMM tasks at revision TSN 86: possible causes, referenced tasks, tools (in store?), expendable parts with IPC P/N (in stock?), steps |
| **Ask** | all | Questions by text or voice (mic button), answered with sources; "Read aloud" |
| **Logbook** | engineer | Checks a draft logbook entry against rules LOG-01..07 before signing; optional AI deep check |
| **Handover** | engineer, lead | Drafts the end-of-shift handover from the shift's records; edit and save |
| **Readiness** | shift lead | Parts / tools / manuals status per aircraft |
| **Coach** | trainee | Prepare a real defect; graded against the verified dossier with guiding questions |
| **Source** | all | Any evidence chip opens the AMOS record or the evidence held in the brief |

### Demo script (the four moments from the brief)

1. **Shift brief:** Tonight as Engineer A. A6-AQP is pinned: deferral limit 20 Oct 2026 passed (WO 6204647).
2. **Aircraft pack before landing:** Aircraft → station clock 21:00: A6-AQP pack ready; 01:00: more packs appear.
3. **Question with the exact manual page:** Ask (type or speak) "What does the TSM say for BRAKES HOT on A6-AOJ?"
4. **Logbook entry checked and handover:** Logbook → Check entry (gaps flagged); Handover → Draft → edit → Save.
   Bonus: Me → Trainee → Coach.

## 8. How it works

### Agents (per aircraft, in parallel)

| Agent | Does | AI? |
|---|---|---|
| Orchestrator | plans the shift, runs the agents, ranks (critical deadlines always first), pins the top item | no |
| Compliance | deferral / MEL deadlines from AMOS action text (latest extension wins) | no |
| Triage | groups AVIATAR messages into episodes; REP-01 repetitive rule; sets aside fleet-wide messages (e.g. GPS/ADS-B on 80+ of 97 tails) | no |
| Manuals | fault index → TSM task, fleet memory (tasks engineers used on similar defects), meaning-based search; effectivity by MSN | no |
| Historian | tail and fleet history, labelled by whether the fault came back in AVIATAR after the fix | no |
| Logistics | TSM → AMM → tools and IPC part numbers (incl. alternatives) → station stock and tool store | no |
| Review layer | R1 schema, R2 every sentence cites evidence, R3 guardrail linter + independent Claude critic | yes |
| Ask / Coach / Handover | tool-using Q&A; Socratic coaching; cited handover summary | yes |

### Retrieval

Exact keys first (fault messages, task numbers, part numbers), then fleet case memory, keyword (Postgres FTS),
trigram and meaning-based (BGE-small embeddings in pgvector) rankings, merged with reciprocal rank fusion,
filtered by effectivity and checked for sufficiency. When nothing is confident it says so instead of guessing.

### Data: real and synthetic

* **Real (data pack):** AMOS work orders, stores, tools; AVIATAR faults and PHA cases; TSM, AMM, IPC (TSN 86,
  01 Nov 2026); the fleet allocation table.
* **Synthetic (not in the data pack, labelled in the UI):** flight schedule, stands, engineer roster, handover
  notes. The clock is fixed at 20 Nov 2026 21:00 (the data ends 20 Nov 06:10).

### Guardrails and audit

No write path to AMOS; no sign/defer/release actions in the UI; every flag names its rule; every reference shows
its revision; every review, Ask, check, pack and handover is written to a hash-chained audit log
(`curl localhost:8000/api/audit` verifies the chain).

## 9. Accuracy and tests

Backtest: 587 closed defects whose actions cite the TSM task the engineer used. Retrieval sees only history from
before each defect and never the defect itself.

| Retrieval | recall@1 | recall@3 | precision@3 | coverage |
|---|---|---|---|---|
| fault index + text search (v1) | 7.7% | 14.5% | 25.1% | 58% |
| + fleet case memory | 28.3% | 40.4% | 47.6% | 85% |
| + effectivity | 28.4% | 40.9% | 49.2% | 83% |
| **+ meaning-based search (current)** | 28.3% | **45.0%** | **51.5%** | **87%** |

```bash
docker compose exec app python -m pytest -q tests                        # rule unit tests (10)
docker compose exec app python -m evals.backtest_tsm 600                  # retrieval backtest (no AI tokens)
docker compose exec -e USE_DENSE=0 app python -m evals.backtest_tsm 600   # without embeddings
cd web && npm install && npm run typecheck                                # UI type check
```

## 10. Deploy to Kubernetes (2 GB disk)

The server only serves. Build the knowledge base on a laptop ([section 5](#5-create-the-knowledge-base-from-the-data-pack))
and stream it in; the data pack and the `.dump` file never go to the server. Summary:

```bash
# laptop: images
docker build --target runtime -t $REG/lineready-api:$TAG .
docker build -t $REG/lineready-web:$TAG ./web
docker build -t $REG/lineready-db:$TAG ./deploy/postgres
docker push $REG/lineready-api:$TAG; docker push $REG/lineready-web:$TAG; docker push $REG/lineready-db:$TAG
# laptop: cluster
kubectl apply -f deploy/k8s/namespace.yaml
kubectl -n lineready create secret generic lineready-secrets --from-literal=...   # see deploy/README.md
kubectl apply -k deploy/k8s
./scripts/restore_kb.sh kb/lineready-kb-<date>.dump --k8s lineready
```

Server footprint ≈ 1.75 GB (images 296 + 307 + 396 MB, database ~740 MB). Full guide: **`deploy/README.md`**.

## 11. Configuration

| Variable | Where | Default | Meaning |
|---|---|---|---|
| `ANTHROPIC_API_KEY` | `.env` / secret | empty | Claude; empty = template mode |
| `LLM_MODEL` | `.env` / ConfigMap | `claude-opus-5-5` | model for all AI calls |
| `LLM_NARRATE_TOP` | `.env` / ConfigMap | `5` | AI summaries per brief |
| `DEMO_NOW` | `.env` / ConfigMap | `2026-11-20T21:00:00` | the brief's "now" (data-pack replay) |
| `STATION` | `.env` / ConfigMap | `SHJ` | station label |
| `LINEREADY_API_URL` | UI (`web/.env.local`, compose, k8s) | `http://localhost:8000` | where the UI's proxy sends `/backend/*` |
| `EMBED_THREADS` | build machine | `4` | embedding threads (4 measured fastest) |
| `USE_DENSE`, `USE_EFFECTIVITY` | evals | `1` | switch features off for comparisons |
| `LINEREADY_NO_MODEL` | testing | unset | `1` = behave like the server (precomputed vectors only) |

## 12. Project layout

```
lineready/
├── lineready/                 backend (Python)
│   ├── domain/                model + rules: deadline, repetitive, effectivity, policy linter, registry
│   ├── application/           orchestrator, agents (triage, specialists, review, ask, coach, logbook),
│   │                          pack.py (aircraft packs), handover.py, seed_shift.py
│   ├── retrieval/search.py    exact / fleet memory / FTS / trigram / dense, RRF, effectivity, AMM/IPC lookups
│   ├── infrastructure/        sources (AMOS, AVIATAR, TSM, AMM/IPC), Postgres, audit chain, Claude, embeddings
│   └── interfaces/            api.py (FastAPI), cli.py, web/ (old debug UI)
├── web/                       tablet UI (Next.js 15, React 19); adapter in src/lib/backend.ts; see web/README.md
├── scripts/                   build_kb.sh (laptop), restore_kb.sh (local or --k8s)
├── deploy/                    k8s/ manifests (Kustomize), postgres/ slim image, README.md
├── evals/                     backtest_tsm.py
├── tests/                     rule unit tests
├── docs/                      architecture document
├── Dockerfile                 --target build (laptop) / runtime (server, default)
└── docker-compose.yml         db (5433), app (8000), web (3000)
```

## 13. Troubleshooting

| Symptom | Fix |
|---|---|
| `localhost:3000` shows "Could not load the brief" | API not ready: `curl localhost:8000/api/readyz`. If "TSM not ingested", load the knowledge base (section 4 or 5). |
| `/api/readyz` returns 503 | Database empty or still starting; run `restore_kb.sh` or `build_kb.sh`. |
| Port 3000 or 8000 already in use | Stop the other process, or change the left side of `ports:` in `docker-compose.yml`. |
| Embedding step very slow | Keep `EMBED_THREADS=4`; the default thread count oversubscribes the CPU (~15x slower). |
| Briefs show "template" summaries | No `ANTHROPIC_API_KEY`, or the app was not recreated after adding it (section 6). |
| Voice button disabled | The browser has no speech recognition (use Chrome/Edge) or no microphone permission. |
| `restore_kb.sh` checksum fails | The file was changed or cut during copy; copy it again with its `.sha256`. |
| Permission denied on `var/` | It was created by a root container; `sudo chown -R $USER var`, or leave it (only containers write there). |

## 14. Known limits

* No login yet: put the Ingress behind corporate SSO before real users.
* REP-01 / REP-02 thresholds must be confirmed against the company MOE/TPM.
* Schedule, roster and handover inputs are synthetic (not in the data pack).
* On the server, free-text Ask questions use keyword and fleet search (no embedding model there).
* Voice uses the browser's speech service; switch to on-premise speech for production.
* Similar-defect lookup takes ~1.3 s (keyword part over all work orders); fine for briefs, can be optimised.
