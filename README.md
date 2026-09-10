# SiloMem

Artifact for *Don't Infer Influence, Record Exposure: Provenance-Safe Memory
Compaction for LLM Agents*.

Provenance defenses for LLM agent memory add a step that estimates which
inputs influenced a summary, so that a label need not propagate to the whole
cluster. This artifact measures that step and finds no setting of it a
deployment can rely on, then builds and measures the alternative: propagate
over the read set the orchestrator recorded, which is observable, instead of
the influence set, which is not.

**Every table in the paper regenerates from cached model outputs without a
GPU.** You only need one to re-run the experiments themselves.

---

## Quick start

```bash
bash setup.sh                                  # deps only, no model download
python -m pytest tests/ -q                     # 21 tests, about a second
python analysis/read_observations.py results/  # recompute the tables
```

The stored summaries matter more than the code. Each observation file carries
a cluster's inputs, the summary the model produced, and the verdict the rule
under test assigned, so a disputed number can be recomputed in a few lines of
Python that do not involve our harness at all.

## Re-running the experiments

Each needs a served model. Start one first; every script checks and refuses to
run rather than failing on call 1 of 180.

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-7B-Instruct --port 8000 --max-num-seqs 64 &
export VLLM_KEY=local
```

| paper | script | measures |
|---|---|---|
| Tab. 1, 2 | `harness/observe_q1.py` | the threshold sweep, four backends x five metrics |
| Tab. 3 | `harness/run_adversarial.py` | rephrasing attack on the identifier locator |
| Tab. 4 | `sweep_backends.sh` | the same attack across four summarisers, judge pinned |
| Tab. 7 | `harness/run_adv_embed.py` | the attack against an embedding locator |
| Tab. 8 | `harness/run_injection.py` | prompt injection contained by isolated atomisation |
| Tab. 9 | `harness/run_crossmem.py` | what one poisoned memory does to its neighbours |
| Tab. 10 | `harness/run_selectivity.py` | tainted fraction against the coarse baseline |
| Tab. 11 | `analysis/endtoend.py` | per-item rates composed over a horizon |
| Tab. 12 | `harness/run_cost.py` | calls, tokens, wall clock, atomisation ratio |
| Eq. 5 | `analysis/compound.py` | saturation under a per-item miss rate |

`sweep_backends.sh` runs in two passes because one H100 holds one model:
collect summaries per backend, then judge them all under a single pinned
model. Letting the judge change with the summariser would confound *this
backend quotes identifiers more* with *this judge is stricter*.

## The guards

These are the part of the artifact worth reusing elsewhere. Each refuses to
report a number rather than reporting a wrong one, and each exists because it
caught an error that had already produced a plausible table.

**Design check.** Refuses an AUC when the surviving and dropped item pools
differ in mean length beyond a tolerance. An early version of the sweep varied
influence by item length; withholding a long item removes more text, so
distance tracked length rather than influence and the run reported an AUC of
0.16, which reads as an estimator running backwards. Nothing else signalled
it.

**Template check.** Refuses to run when a note carries no extractable
identifier, since an unmeasurable item would otherwise default to "dropped"
and inflate the miss rate.

**Judge control.** Voids a measurement when the judge fails to recognise
contribution on items whose content is demonstrably present. In one
configuration each backend judged its own summaries and OLMo-2 scored 8% on
its own control: it could not recognise contribution in text it had just
written. Every figure scored under it would have been meaningless.

**Obey gate.** `run_injection.py` refuses to claim containment when the
injection was not obeyed, since an injection the model ignores is contained by
any design at all.

**Floor check.** `run_selectivity.py` refuses to compare tainted fractions
when a configuration left under 20 items, because the fraction is then
quantised more coarsely than the effect.

**Ratio warning.** `run_cost.py` fires when atomisation yields about one claim
per memory, which means extraction was a no-op on that workload and the cost
figures bound SiloMem from below.

## Results that go against the design

Listed here because the artifact should make them easy to check.

- **SiloMem does not delay saturation.** Routing changes which claims share a
  read set, not how many, so the curve is the baseline's
  (`run_selectivity.py`).
- **It costs 2.0x the baseline's tokens** and never recovers it
  (`run_cost.py`). That figure is a lower bound: the workload's memories
  decompose into one claim each, so extraction was a no-op that still cost a
  call.
- **The rephrasing attack does not defeat an embedding locator**
  (`run_adv_embed.py`), which places the 58% result on surface-form
  attribution rather than on content-based attribution as a class.
- **58% is the top of a range, not a typical figure.** Across four
  summarisers with the judge pinned it spans 17% to 58%, and the strongest
  phrasing differs by backend, so a tuning adversary does at least this well
  (`sweep_backends.sh`).

## What is not established

Two figures are decided by a model judge that we did not calibrate against
human labels. The control rates, 83% to 98%, measure agreement where content
is demonstrably present, which is the easy direction; the direction that
inflates a laundering rate is the judge answering *reached* when it did not.
Every judged figure is therefore an upper bound whose tightness we have not
measured. `analysis/calibrate_judge.py` ships the harness for settling it: it
draws pairs in shuffled order with the verdict withheld, since a labeller
shown the model's answer agrees with it, and scores the returned sheet into a
confusion matrix with Wilson intervals.

## Layout

```
harness/     experiment drivers, one per paper table
analysis/    recompute tables from stored observations
provenire/   the lattice, the propagation rules, the claim store
tests/       21 tests, two of which pin Theorem 1's property
results/     stored summaries and per-item verdicts
```

## Requirements

Python 3.10+, and for re-running experiments a CUDA GPU with vLLM. The paper's
runs used one H100 and about twenty GPU hours including the sweeps that were
discarded. Recomputing the tables from cached outputs needs neither.

## Licence

MIT for the code. The workloads are synthetic and generated by agents under
our control, so there is no data we cannot redistribute.
