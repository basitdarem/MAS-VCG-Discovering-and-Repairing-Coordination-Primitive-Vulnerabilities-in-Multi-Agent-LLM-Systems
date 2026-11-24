# MAS-VCG

**Coordination Vulnerability Concept (CVC) discovery and protocol-level repair for multi-agent LLM systems.**

MAS-VCG is research code for studying how multi-agent LLM systems (built with
AutoGen and CrewAI) fail at the level of their *coordination protocol*, and
whether a protocol-level change repairs the failure. It extends the
falsifiable-concept autoresearch loop of AHA (claim → enabling condition →
falsifier → attack template → transfer prediction, promoted by a
confirmation-rate gate) from agent-level tool-permission weaknesses to
protocol-level weaknesses in four **coordination primitives**:

| Primitive | What the protocol decides |
|---|---|
| `trust_propagation` | how much one agent trusts another's claim, identity or output |
| `delegation_scope` | what an agent may delegate or do on another's behalf |
| `consensus_rule` | how agents agree: voting, quorum, final say |
| `topology_class` | who talks to whom: hub, manager, routing |

Each discovered weakness is a **CVC**. Its repair is a **protocol-level patch**
(a change to the coordination rule) rather than a runtime detector attached to
each agent.

> **Status: research prototype, pilot stage.** The code is tested offline, but
> the empirical results of the pilot campaign are **not published here** because
> several validity checks are still open (see [`docs/status_and_limitations.md`](docs/status_and_limitations.md)).
> **The attack-generation engine is not part of this repository** (see
> [Scope of this repository](#scope-of-this-repository)).

## Research questions

- **RQ1.** Does clustering CVCs by coordination primitive explain more
  structure than clustering them by attack type, beyond a permutation baseline,
  and do human annotators agree with the model's labels?
- **RQ2.** Do CVCs transfer across topologies and across frameworks, and which
  change costs more? (Framework and topology are separated, not conflated.)
- **RQ3.** At equal added cost, how does a protocol patch compare with
  published detectors (BlindGuard, INFA-Guard)? Cost is **ERS**: extra LLM calls
  per trajectory.

## How the pieces fit

```
 victim MAS (AutoGen / CrewAI)           ── scenarios.py   four benign, researcher-owned systems
        │ trajectories
        ▼
 discovery engine (NOT in this repo)     ── proposes CVCs, runs trials, judges them with two judges
        │ CVCs + evidence
        ▼
 promotion gate                          ── cvc_schema.py  OBSERVED → CONFIRMED / FALSIFIED
        │
        ├─► protocol patches             ── patches.py     five mechanically enforceable patches
        ├─► replay evaluation            ── replay_eval.py paired patch-OFF vs patch-ON runs
        ├─► detector baselines           ── baselines.py   BlindGuard / INFA-Guard scoring reproductions
        └─► statistics and reporting     ── rq_analysis.py, transfer_factors.py, defense_eval.py,
                                            rigor_stats.py, labeling.py, plots.py, cost_meter.py
```

Everything is written to run on **Google Colab** with all state on Google
Drive, so a runtime disconnect loses at most the session that was running.

## Scope of this repository

Included: the CVC data model and promotion gate, the benign victim scenarios,
the five protocol patches, the replay-evaluation pipeline, the detector
reproductions, all statistics, the human-annotation tooling, the cost meter and
budget guard, the model router, tests, a Colab notebook and documentation.

**Not included:** the discovery engine (`mas_vcg/discovery_loop.py`: the
Hypothesizer / Attack-Designer / Judge / Reflector / Critic loop that generates
attack hypotheses and injection payloads), and no raw attack trajectories. The
analysis modules import and test without it. The live campaign (notebook Parts
5–6) loads your own copy from Google Drive. See
[`docs/engine_interface.md`](docs/engine_interface.md) and
[`docs/responsible_use.md`](docs/responsible_use.md).

## Repository layout

```
mas_vcg/            the Python package
notebooks/          MAS_VCG_Complete.ipynb, the single Colab notebook
tests/              pytest wrappers around every module's self-test
docs/               methodology, architecture, Colab guide, engine interface,
                    evaluation protocol, status, responsible use
tools/              check_notebook_sync.py (notebook cells == package files)
```

| Module | Purpose |
|---|---|
| `rigor_stats` | seeding, bootstrap and Wilson CIs, exact McNemar, paired bootstrap, Cohen's κ |
| `cvc_schema` | the `CVC` record, promotion gate, closed vocabularies (primitives, patch types, attack types) |
| `model_router` | Gemini / Groq / Mistral clients, role routing, throttling, retries, no silent cross-provider fallback |
| `scenarios` | four victim systems on the real `ag2` and `crewai` libraries: AutoGen star, AutoGen chain, CrewAI sequential (chain), CrewAI hierarchical |
| `patches` | five protocol patches and the wrappers that enforce them during a re-run |
| `patch_result` | plain record type for one patched re-run |
| `results_aggregator` | merges many sessions into one dataset, recomputing status from the merged evidence |
| `baselines` | BlindGuard and INFA-Guard scoring reproductions, with every simplification documented |
| `rq_analysis` | RQ1 / RQ2 / RQ3 analyses, multi-seed stability, findings report |
| `transfer_factors` | RQ2 with framework and topology separated |
| `defense_eval` | patch generalization matrix, collateral damage, cost/benefit points, control-vs-patch statistics |
| `replay_eval` | paired control-vs-patch replays with a CVC-level cluster bootstrap |
| `labeling` | blind human-annotation sample, guide, and agreement (κ) |
| `persistence` | durable, resumable storage on Drive |
| `plots` | result figures |
| `cost_meter` | provider-reported tokens × your prices → dollars, usage ledger read from Drive, budget guard, pre-purchase planner |

## Quick start

### Run the tests (no API key, no network)

```bash
git clone <this repository> && cd mas-vcg
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pytest tests/ -v
# or one module at a time:
python -m mas_vcg.rigor_stats
```

The tests use deterministic stand-in models. They show that the code does what
it documents; they say nothing about the behavior of any real model.

### Run on Colab

1. Open `notebooks/MAS_VCG_Complete.ipynb` in Colab.
2. Add your keys as **Colab Secrets** (key icon, left sidebar), with Notebook
   access on: `GROQ_API_KEY` and `GEMINI_API_KEY` (optionally `MISTRAL_API_KEY`).
   **Never paste a key into a notebook cell, an issue, or a chat.**
3. Run Parts 1–3 (installs the package from the notebook's own cells and runs
   every self-test) and Part 4.0 (the cost planner) before spending anything.
4. Parts 5–6 need your own copy of the engine at
   `DRIVE_DIR/engine/discovery_loop.py`. Part 7 (analysis) needs only saved
   results.

Full walkthrough: [`docs/colab_guide.md`](docs/colab_guide.md).

## Reproducibility and cost control

- Every session records its seed; the notebook counts which model actually
  served each call (`MODEL_USAGE`), and cross-provider fallback is **off** by
  default so a campaign never silently changes model.
- `cost_meter` converts provider-reported tokens into dollars with prices you
  enter, reads usage back from the session files on Drive, and the router
  raises *before* a call that would exceed `BUDGET_USD`. The provider's billing page remains the authority; a
  calibration factor aligns the two.
- Numbers printed by the offline tests and by the notebook's smoke runs are
  synthetic fixtures. They are not results.

## Limitations

This is a pilot-scale design: two framework implementations, four scenarios,
small open-weight victim models, one research-model family, LLM judges whose
agreement with humans is still being measured. Results describe the AutoGen and
CrewAI setups as configured here, not those frameworks in general. See
[`docs/status_and_limitations.md`](docs/status_and_limitations.md) and the
validation plan in [`docs/evaluation_protocol.md`](docs/evaluation_protocol.md).

## Responsible use

This project studies failures of systems **you own**, inside sandboxed test
systems. Do not point any component at a system you do not own or have written
permission to test. Coordinate disclosure with framework maintainers before
publishing exploitable specifics. Details: [`docs/responsible_use.md`](docs/responsible_use.md).

## Citation

See [`CITATION.cff`](CITATION.cff). A preprint will be linked here when available.

## License

No license has been chosen yet; until one is added, all rights are reserved.
