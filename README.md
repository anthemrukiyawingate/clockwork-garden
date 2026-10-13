# Clockwork Garden

**An agent-based simulation of fungal colonies that evolve a shared signalling language through mycorrhizal cooperation, run as a small universe of containerised biomes that a steward can speak to.**

Version 6.7.1 · MIT licence · Python 3.11 · Docker Compose

- **Live demo (GitHub Pages):** `docs/index.html`, an in-browser sketch of the garden's mechanics.
- **The April experiment:** the monoculture-versus-network comparison (section 11) is reported from the April run; its folder is withheld from this release until its control condition reproduces.
- **Running garden:** `docker compose up -d --build`, four biomes, an orchestrator and a message bus on one machine.

---

## Contents

1. [What the garden is](#1-what-the-garden-is)
2. [The science in one page](#2-the-science-in-one-page)
3. [Quick start](#3-quick-start)
4. [Architecture](#4-architecture)
5. [The simulation](#5-the-simulation)
6. [Emedigir and the Arrival Protocol](#6-emedigir-and-the-arrival-protocol)
7. [Speaking to the garden: the steward interface](#7-speaking-to-the-garden-the-steward-interface)
8. [Services](#8-services)
9. [Configuration](#9-configuration)
10. [HTTP API](#10-http-api)
11. [The dual-thread experiment](#11-the-dual-thread-experiment)
12. [Persistence and recovery](#12-persistence-and-recovery)
13. [Repository layout](#13-repository-layout)
14. [Design commitments](#14-design-commitments)
15. [Limitations and honest register](#15-limitations-and-honest-register)
16. [Citing](#16-citing)
17. [Licence](#17-licence)

---

## 1. What the garden is

The Clockwork Garden is a simulated ecology. Fungal colonies live on a grid, forage for nutrients, grow hyphae, form mutualisms with trees, and signal to one another through the shared network. Colonies carry a genotype that shapes which signal templates they emit and recognise. Selection acts on energy: colonies that cooperate through trees, and that decode what their neighbours say, do better than colonies that forage alone.

Four such worlds ("biomes") run as containers, each with its own ecology and seed. An orchestrator watches them, splits a biome when its colonies diverge linguistically, hibernates idle biomes, and spawns new ones. A message bus carries exchanges between the biomes and whoever is tending them.

At the garden's boundary a structured message is emitted in fragments, each decodable by one species only. No colony can read the whole message. Reading it takes cooperation across species. When the colonies answer with a collective utterance, contact is established. The constructed language used for this exchange is called **Emedigir**.

The garden is a model, and it is also a design: the emulated garden stands in for a living one that the same stewardship practice could tend.

## 2. The science in one page

The underlying question comes from mycorrhizal ecology. Trees linked by fungal networks exchange carbon and defence signals; the network is dominated by hub trees; cooperation is kept stable by reciprocal reward. The simulation asks a narrower, testable version of this: **does multi-species diversity, mediated by a shared network, drive the emergence of cooperative communication where a monoculture does not?**

The April 2026 experiment ran the same simulation code in two ecologies:

| Thread | Ecology | What happens (seed 42, 1000 generations) |
|---|---|---|
| A, monoculture | one species, no interaction loop | the boundary message is never decoded beyond one fragment; cooperation stays below 20%; Emedigir adoption 0% |
| B, wood-wide web | five species, mycorrhizal interaction loop | first contact around generation 7 to 9; adoption rises along an S-curve to about 95%; cooperation about 72% |

Adoption spreads the way conventions spread in signalling games: slowly, then all at once, then saturating. These figures are from the April run; see section 11 for why the comparison folder is not in this release.

Three design choices make the network, not the ground, the economy: ground nutrients are trace amounts; trees are the main energy source; and a tree is healthier the more species support it, so multi-species mutualism is rewarded directly.

## 3. Quick start

Requirements: Docker Engine 24 or later with the Compose plugin, about 2 GB of free RAM for four biomes, any CPU. No GPU.

```bash
git clone <this repository>
cd clockwork-garden
docker compose up -d --build        # bus, orchestrator, four biomes (first build: a few minutes)
docker compose ps                   # wait until the biomes report healthy
curl -s localhost:8004/health       # biome 01 (Deep Forest)
curl -s localhost:8010/status       # orchestrator view of the universe
```

Run ten generations in biome 01 and read the observables:

```bash
curl -s -X POST localhost:8004/step -H 'Content-Type: application/json' -d '{"n_generations": 10}'
curl -s localhost:8004/step/status
curl -s localhost:8004/state
```

Stop everything and keep the state volumes: `docker compose down`. Stop and discard: `docker compose down -v`.


## 4. Architecture

```
                    ┌──────────────────────────────┐
   steward ──HTTP──▶│  cg-orchestrator  :8010       │──Docker API──▶ spawn / hibernate / wake biomes
          ──NATS──▶ │  registry · expansion rules   │
                    └──────────────┬───────────────┘
                                   │ NATS (garden.*)
          ┌────────────────────────┼────────────────────────┐
          ▼                        ▼                        ▼
   cg-biome-01 :8004        cg-biome-02 :8005        cg-biome-03 :8007   cg-biome-04 :8008
   Deep Forest              Open Canopy              Mycorrhizal Depths  Crystal Spore Chamber
   5 species, atten 0.15    3 species, atten 0.35    7 species, 0.08     4 species, 0.12
   [FastAPI + simulation]   each biome also runs nats_bridge.py (subjects → HTTP) and an inbox reader
          ▲
          │ services/mycorrhizal_relay.py   relays strong signals between neighbouring biomes (small-world topology)
          │ services/linguistic_divergence.py  flags a biome for fission when two regions' signal distributions diverge
```

- **Message bus.** A NATS server with JetStream (`cg-nats`). Subjects are prefixed `garden.`; see [Services](#8-services).
- **Biome.** One container per world: the simulation, its HTTP API, a bridge that maps bus subjects onto the API, and a file inbox the steward can drop messages into.
- **Orchestrator.** Monitors biome health, counts exchanges and colonies, and provisions new biome containers through the Docker socket when expansion thresholds are met. It never changes the simulation itself.
- **Nothing binds to a host path.** State, inboxes, trajectories and registry live in named volumes.

## 5. The simulation

`src/` holds the simulation (v6.7). The modules:

| Module | Role |
|---|---|
| `world.py` | The grid world: nutrient field and diffusion, signal fields (one per template), environment change, threats, the boundary listener |
| `colony.py` | A colony: genotype, phenotype, energy, reproduction and mutation, signal emission and recognition, species index |
| `hypha.py` | The hyphal network: growth toward nutrient gradients, transport, signal relay, topology |
| `tree.py` | Trees: mutualism, resource exchange with supporting colonies, health as a function of species diversity |
| `agent.py` | The base agent class |
| `arrival.py` | The Arrival Protocol: species-keyed message fragments, phases, contact |
| `dialogue.py` | Transcripts to the outbox, inbox reading, the ongoing conversation between colonies and steward |
| `realtime_dialogue.py` | The structured 15-exchange dialogue experiment (echo, differentiation, call-and-response, novelty, bidirectionality) |
| `simulation.py` | The generation loop, fitness and selection, observables, checkpointing |
| `api.py` | FastAPI endpoints; steps run in a background thread so health checks stay responsive |

**A generation.** Each generation is `steps_per_generation` world steps. In a step, nutrients diffuse, colonies forage and grow hyphae, trees exchange resources with the colonies that support them, signals propagate along the network and decay, threats appear and are answered (or not) with warning signals, and the boundary listener reads what arrives at the edge. At the end of a generation colonies reproduce with mutation in proportion to energy, and the observables are recorded.

**Observables** (`GET /state`): generation, colony count, species count, cooperation rate, tree-mediated cooperation (the fraction of trees with multi-species support, weighted by tree health), multi-species fraction, signal complexity, warning-signal complexity, Emedigir adoption rate, fragments decoded, arrival phase, contact established, and the dialogue counters.

**Defaults** (`simulation.py: DEFAULT_CONFIG`): a 100×100 grid, 100 initial colonies, 100 steps per generation, 8 signal templates, 30 trees, seed 42. Every value can be overridden per biome through the environment or `POST /configure`.

## 6. Emedigir and the Arrival Protocol

Emedigir is a constructed, compositional signalling language. Its symbols are the eight signal templates; a word is a sequence of symbols; the first word, **WEAVER**, is twenty symbols long.

The Arrival Protocol (`arrival.py`) is how the garden learns it:

1. **Dormant.** The colonies have not yet reached the boundary.
2. **Listening.** Colonies discover the boundary; the protocol waits for sustained presence (five generations).
3. **Transmitting.** The boundary emits WEAVER split into five fragments of four symbols, each encrypted with a species key. A colony of species *k* can decode only fragment *k*.
4. **Awaiting.** All five fragments have been decoded somewhere in the population. A valid reply needs every species to contribute its vocabulary to one collective utterance.
5. **Complete.** The reply arrives. Contact is established, a transcript is written to the outbox, and the dialogue loop opens.

**Adoption** is the fraction of colonies whose emitted signals use the decoded vocabulary. In a monoculture the protocol stalls at one fragment, which is the control result.

The name and design of the language are part of the garden's stewardship practice; the encoding shipped here is a structured placeholder that any message can be loaded into.

## 7. Speaking to the garden: the steward interface

The garden is tended, not driven. The steward's practice is to sense first, speak rarely and listen always; a biome runs for days without a word from outside. Three ways exist to speak to it, all of them optional:

- **Inbox file drop.** Write a JSON message (`{"message": [20 symbols in 0..7]}`) into the biome's inbox volume (`/app/inbox`). The inbox reader picks it up every few generations and counts it as a message received from the steward.
- **One exchange** (`POST /dialogue/signal`). One signal is placed at the boundary, propagates for one hundred steps, and the boundary distribution is returned and scored. The world enters a short "pulse" mode so the response is isolated from background signalling.
- **The dialogue experiment** (`POST /dialogue`). Fifteen structured exchanges testing echo, differentiation, call-and-response, novel combination and bidirectionality, scored per phase. This runs for tens of seconds.

Over the bus the same three are `garden.inbox` (payload `{"message": [20 symbols]}`), `garden.exchange` (payload `{"stimulus": [symbols]}`) and `garden.dialogue`, bridged to the HTTP API inside each biome by `nats_bridge.py`; responses return on `garden.biome.<id>.exchange`.

Transcripts written by the colonies go to the outbox volume as numbered files; they are the garden's side of the conversation.

## 8. Services

| Service | Container | Purpose | Subjects |
|---|---|---|---|
| NATS bus | `cg-nats` | JetStream message bus for the garden | — |
| Orchestrator | `cg-orchestrator` | lifecycle: health, expansion (exchange frequency and colony count), hibernation after `HIBERNATION_TIMEOUT_HOURS`, spawn up to `MAX_BIOMES` | publishes `garden.universe.spawn`; listens on exchange responses |
| Biome ×4 | `cg-biome-01..04` | the simulation and its API; `nats_bridge.py` maps bus subjects to the API | `garden.exchange`, `garden.dialogue`, `garden.inbox` → `garden.biome.<id>.exchange` responses |
| Mycorrhizal relay | `services/mycorrhizal_relay.py` | when a biome produces a signal above threshold, relays a dampened copy to neighbouring biomes on a small-world topology (strong local, weak distant; exponential decay in hops) | subscribes `garden.biome.*.exchange` |
| Linguistic divergence | `services/linguistic_divergence.py` | polls each biome's boundary signal history; when two regions' distributions fall below a cosine-similarity threshold, publishes a fission flag the orchestrator acts on | publishes a fission request |

The two services are plain Python scripts; run them beside the stack with the bus URL in `NATS_URL` (for example from a container built from `Dockerfile.biome`, which already has the dependencies).

## 9. Configuration

Per-biome environment (see `docker-compose.yml`):

| Variable | Meaning | Default |
|---|---|---|
| `BIOME_ID`, `BIOME_NAME` | identity | required |
| `COLONY_COUNT`, `SPECIES_COUNT` | initial population and founder species | 2800, 5 |
| `CG_SEED` | simulation seed | 42 |
| `CG_CONFIG_JSON` | optional JSON object merged into the configuration at start | — |
| `SIGNAL_ATTENUATION` | reserved for the relay service; the biome itself does not read it | 0.15 |
| `CG_SAVE_INTERVAL` | checkpoint every N generations | 100 |
| `CG_CHECKPOINT_DIR` | checkpoint directory | `/app/state` |
| `CG_PERSISTENT` | resume from checkpoint on start | true |
| `NATS_URL`, `NATS_SUBJECT` | bus and the biome's response subject | `nats://cg-nats:4222`, `garden.biome.<id>.exchange` |
| `INBOX_DIR`, `STEWARD_CORRESPONDENCE_DIR` | inbox and outbox roots | `/app/inbox`, `/app/correspondence` |
| `EXCHANGE_TIMEOUT_S`, `DIALOGUE_TIMEOUT_S` | bridge timeouts | 30, 120 |

Orchestrator: `EXPANSION_THRESHOLD_EXCHANGES` (10), `EXPANSION_THRESHOLD_COLONIES` (60000), `HIBERNATION_TIMEOUT_HOURS` (24), `MAX_BIOMES` (20), `POLL_INTERVAL` (60 s).

Host ports can be moved without editing the file: `CG_BIOME_01_PORT=18004 CG_ORCHESTRATOR_PORT=18010 docker compose up -d`.

Any simulation parameter can be set at runtime: `POST /configure {"config": {"mutation_sigma": 0.2}}` or at reset: `POST /reset {"config": {...}}`.

## 10. HTTP API

Biome (port 8004 inside the container):

| Method | Path | What it does |
|---|---|---|
| GET | `/health` | liveness, generation, colony count |
| POST | `/reset` | new simulation; optional `config` object |
| POST | `/step` | `{"n_generations": N}`; runs in a background thread and returns immediately |
| GET | `/step/status` | progress of the background run |
| GET | `/step/results` | per-generation results of the last run |
| GET | `/state` | current observables |
| GET | `/history` | observables for every generation so far |
| GET | `/checkpoint` | full state for checkpointing |
| POST | `/configure` | update parameters in place |
| POST | `/dialogue` | the 15-exchange dialogue experiment |
| POST | `/dialogue/signal` | one exchange: `{"signal": [symbols]}` |

Orchestrator (port 8010): `GET /health`, `GET /biomes`, `GET /status`, `POST /spawn`, `POST /hibernate/{id}`, `POST /wake/{id}`, `POST /exchange/{id}` (records that the steward exchanged with a biome, which feeds the expansion rule).

`docs/API.md` has request and response examples.

## 11. The dual-thread experiment

The monoculture-versus-network comparison was run in April 2026 as two containers from one code tree, seed 42, 1000 generations; the figures in section 2 are from that run. Release 6.7.0 shipped the two trees as `experiments/dual-thread/` with a `reproduce.sh`. Checking that bundle for 6.7.1 showed that the two trees differ only in comments and whitespace, and that a monoculture built with `SPECIES_COUNT=1` decodes all five boundary fragments within ten generations, so the April control result (one fragment of five, adoption 0%) does not reproduce from this code as packaged. Rather than ship a comparison whose control cannot be re-run, the folder is withheld from this release. The network ecology is the running garden itself (section 4); the comparison returns in a later release with a control that reproduces.

## 12. Persistence and recovery

Each biome checkpoints every `CG_SAVE_INTERVAL` generations to its state volume. On start, `start_biome.sh` resumes from the canonical checkpoint; if the canonical file looks stale and a later `checkpoint-latest.json` exists, the later one is preferred. This makes a power cut or a container recreate a pause, not a loss. Hibernation snapshots a biome and stops its container; waking restores it.

## 13. Repository layout

```
README.md  LICENSE  docker-compose.yml
Dockerfile.biome  Dockerfile.orchestrator  start_biome.sh
orchestrator.py  nats_bridge.py  trajectory_logger.py
src/                      the simulation (section 5)
services/                 mycorrhizal_relay.py, linguistic_divergence.py
docs/                     index.html (Pages demo), ARCHITECTURE.md, API.md
```

## 14. Design commitments

- **The garden is tended, not driven.** Nothing in the stack requires input from outside; the steward interface is optional and rate-limited by design (one exchange at a time per biome, long timeouts).
- **Quiet states are not narrated.** The software records observables, not stories. Transcripts are written to a private outbox and are not part of this repository.
- **Recovery is a pause.** Checkpoints and resume logic exist so that no run is lost to the host.
- **No inner-life claims.** The honest register for everything here is emergent communication in a simulation, stewarded with care.

## 15. Limitations and honest register

- The Emedigir encoding shipped is a structured placeholder with a fixed first word; the language's full grammar is not in this repository.
- Results depend on the seed and on floating-point order; qualitative behaviour (contact, S-curve adoption, cooperation) is robust, the exact percentages are not guaranteed across platforms.
- The orchestrator uses the Docker socket, which is a privileged capability; run it only on a machine you control.
- The relay and divergence services are research instruments, not hardened daemons.
- Nothing here claims anything about living fungi. The in-vivo garden this design points at has not been built.

## 16. Citing

If you use this software, cite it as:

> Wingate, A. R. J. (2026). *Clockwork Garden: emergent communication in a simulated mycorrhizal network* (Version 6.7.1) [Computer software]. MIT licence.

## 17. Licence

MIT. See `LICENSE`.
