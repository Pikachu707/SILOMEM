# SiloMem

Artifact for *Don't Infer Influence, Record Exposure: Provenance-Safe Memory
Compaction for LLM Agents*.

Provenance defenses for LLM agent memory add a step that estimates which
inputs influenced a summary, so a label need not propagate to the whole
cluster. This artifact measures that step and finds no setting of it that is
both sound and selective, then builds and measures the alternative: propagate
over the read set the orchestrator recorded, which is observable, instead of
the influence set, which is not.

**Every table in the paper regenerates from cached model outputs without a
GPU.** You need one only to produce new outputs.

---

## Start here

```bash
bash setup.sh                                  # deps only, no model download
python -m pytest tests/ -q                     # 37 tests, about a second
python analysis/read_observations.py results/  # recompute the tables
```

If the tests pass and the tables match the paper, the computational claims
hold without running anything on a GPU.

To dispute a specific judged figure, read the summaries it was scored from:

```bash
python3 -c "
import json; d=json.load(open('results/crossmem.json'))
for s in d['samples'][:3]:
    print('MIXED    :', s['mixed_account'][:200])
    print('SEPARATED:', s['separated_account'][:200]); print()
"
```

That is the level at which a judged number can be checked. The judge is a
generative model and its agreement with a human is unmeasured (Appendix D),
so the stored summaries matter more than our scoring of them.

---

## Reproducing from scratch

Needs one 80GB GPU. On a fresh host:

```bash
bash bootstrap.sh                        # reports what is present, installs what is missing
tmux new -s setup                        # a dropped connection kills an untmuxed run
bash setup_h100.sh 2>&1 | tee setup.log
```

`bootstrap.sh` **does not touch a working NVIDIA driver.** GPU images ship
one, and a version mismatch is harder to recover from than a missing driver.
It also warns below 80GB free: the five backends are roughly 100GB of
weights, and a download that dies at 95% leaves a partial cache that the next
attempt does not resume from.

Then, with vLLM serving:

```bash
export VLLM_KEY=local
python harness/run_selectivity.py --model Qwen/Qwen2.5-7B-Instruct --separated
python harness/run_crossmem.py    --model Qwen/Qwen2.5-7B-Instruct --separated
python harness/run_memgpt.py      --model Qwen/Qwen2.5-7B-Instruct \
    --self-sustaining --summary-tokens 256 --rounds 12 --trials 20
python harness/run_omission.py    --model Qwen/Qwen2.5-7B-Instruct --trials 60
bash sweep_backends.sh                   # five backends, about an hour, mostly loading
```

---

## What maps to what

| Paper | Script | Cached result |
|---|---|---|
| Tab. 2, Fig. 3 — no operating point | `harness/observe_q1.py` | `results/q1_*.json` |
| Tab. 4 — rephrasing attack, five backends | `sweep_backends.sh` | `results/adv_backends.json` |
| Tab. 9, Fig. 6a — cross-memory corruption | `harness/run_crossmem.py` | `results/crossmem.json` |
| Tab. 12, Fig. 6b — selectivity, both routers | `harness/run_selectivity.py --separated` | `results/selectivity.json` |
| Fig. 6a — twelve rounds of recursion | `harness/run_memgpt.py --self-sustaining` | `results/memgpt.json` |
| §4.4 — citation omission | `harness/run_omission.py` | `results/omission.json` |
| Fig. 6c — how p composes | `harness/run_pfrac.py` | `results/pfrac.json` |

---

## The guards, and why they are the most reusable part

Three checks in this harness each caught a table that looked entirely
plausible. They are worth lifting into other work.

**Length confound.** Refuses to report an AUC when the surviving and dropped
item pools differ in mean length beyond a tolerance. Without it a sweep
reported discrimination that was really measuring sentence length.

**Template check.** Refuses a run whose generated items sit too close to their
templates, which would make any attribution trivially correct.

**Judge control.** Scores a batch the judge should unanimously pass. A control
below 85% voids the run instead of producing a number.

A fourth lesson has no guard because it is a reading discipline: **a column
that cannot be non-zero is not measuring anything.** Two such columns reached
draft tables here. One selected items by label and reported how many carried
the other label, which is zero by definition. The other compared against a
module constant that is never empty, so the rule refused on every trial. Both
looked like results.

`VERSION` records every defect found in this harness and what each cost. It
is long on purpose.

---

## What this artifact does not contain

- **A deployed agent.** The workload is synthetic, and the atomisation ratio
  came out at 1.00 because our memories are single sentences, so the 2.0×
  cost figure is a lower bound with the error against us.
- **A calibrated judge.** `harness/judge_ablation.py` is the harness for
  human labelling; we did not run it. Every judged figure in the paper is an
  upper bound of unmeasured tightness.
- **Declassification.** Not implemented. Without it a store only darkens.

---

## Layout

```
harness/     experiment scripts, one per claim
analysis/    regeneration of every table from results/
results/     cached model outputs, enough to recompute every number
tests/       37 tests, no GPU, run them first
VERSION      defect log: what broke, how it was found, what it cost
```

Licensed permissively; see `LICENSE`.
