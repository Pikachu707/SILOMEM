# SiloMem

Artifact for **"Don't Infer Influence, Record Exposure: Provenance-Safe Memory
Compaction for LLM Agents"** (anonymous submission).

LLM agents compact their long-term memory by summarizing groups of entries.
Provenance defenses label each entry with its origin and refuse sensitive
actions that trace back to untrusted content. To keep summaries from being
tainted wholesale, published designs estimate which inputs influenced each
summary. The paper shows that this estimate is neither safe nor selective, and
presents **SiloMem**, which propagates labels over the inputs each model call
was shown (the read set the orchestrator records) and keeps those read sets
small: one memory per extraction call, one label per summary, and an action
check over everything the deciding call saw.

This repository has two parts:

| Folder | Paper | What it contains |
|---|---|---|
| [`compaction/`](compaction/) | Sections 3, 4, 6, Appendix D | Attribution estimators and their failure modes, the SiloMem compaction pipeline, cross-memory corruption, selectivity, cost |
| [`agent/`](agent/) | Section 7, Appendix E | SiloMem inside a tool-using agent on AgentDojo, across sessions, with Qwen2.5-72B transcripts |

---

## Check the results in five minutes (no GPU)

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

(cd compaction && python -m pytest tests -q)   # 37 tests: labels, routing, guarantees
(cd agent && python -m pytest tests -q)        # 27 tests: label tables, guard, memory, agent path
python reproduce.py                             # prints the numbers behind every table and figure
```

`reproduce.py` reads the stored results and prints them under the paper's
table and figure numbers. For Section 7 it recomputes Table 8 from the stored
agent transcripts.

---

## Where each result comes from

| Paper | Result | Stored data | Produced by |
|---|---|---|---|
| Table 2, Figure 2 | Ablation at its loosest safe threshold | `compaction/results/q1_operating.csv` | `compaction/harness/observe_q1.py` |
| Figure 3 | AUC of five metrics on four models | `compaction/results/metrics.csv` | `observe_q1.py`, `analysis/similarity_sweep.py` |
| Section 3.2 | Abstention band | `compaction/results/lo_sweep.csv` | `compaction/harness/observe_embed.py` |
| Table 3 | Rewording attack, four wordings | `compaction/results/adversarial.csv` | `compaction/harness/run_adversarial.py` |
| Figure 4 | Rewording attack, five summarizers | `compaction/results/attackbackend.csv` | `compaction/sweep_backends.sh` |
| Table 5 | Rewording attack against each rule | `adversarial.csv` (identifier row) | `compaction/harness/run_adv_embed.py` (embedding row) |
| Table 6 | Prompt injection inside a memory | | `compaction/harness/run_injection.py` |
| Section 4.4 | Citations versus the recorded read set | `compaction/results/omission.csv` | `compaction/harness/run_omission.py` |
| Figure 6 | Damage to the other memories' records | `compaction/results/crossmem.csv` | `compaction/harness/run_crossmem.py` |
| Section 6.2 | Compaction stage, both routers | `compaction/results/crossmem_compaction.csv` | `run_crossmem.py --separated` |
| Figure 7a | Self-replicating payload, twelve rounds | `compaction/results/memgpt.csv` | `compaction/harness/run_memgpt.py --self-sustaining` |
| Figure 7b | Tainted share after two rounds | `compaction/results/selectivity.csv`, `selectivity_sep.csv` | `compaction/harness/run_selectivity.py [--separated]` |
| Table 7 | Cost of each rule | `compaction/results/cost.csv` | `compaction/harness/run_cost.py` |
| Figure 9 | Trajectories that act on laundered content | `compaction/results/endtoend.csv` | `compaction/analysis/endtoend.py` |
| Table 8 | Attacks through memory, later-task utility | `agent/results/qwen72b/qwen72b_E1.jsonl.gz` | `agent/scripts/run_all.sh` (E1) |
| Figure 8a | Trusted-input tasks by summaries retrieved | `agent/results/qwen72b/summary.md` | `run_all.sh` (E2) |
| Figure 8b | Refused actions shown to a simulated user | `agent/results/qwen72b/summary.md` | `run_all.sh` (E4) |
| Table 11 | Later tasks with no attack | `agent/results/qwen72b/summary.md` | `run_all.sh` (E3) |
| Appendix E | Label tables, trusted-input tasks, payload funnel | `agent/results/qwen72b/E1_diag.txt` | `agent/silomem_agent/labels.py`, `extra_tasks.py`, `agent/diagnose.py` |

The CSV files in `compaction/results/` hold the numbers reported in the paper;
the header of each file states the model, sample size and settings of the run
it summarizes. Each experiment script writes a JSON file with every model
output it used.

---

## Re-running the experiments

Both parts talk to any OpenAI-compatible server. We used vLLM.

### Sections 3, 4 and 6 (one 80 GB GPU)

```bash
cd compaction
bash bootstrap.sh                 # reports what is installed; never touches a working driver
bash setup_h100.sh                # dependencies, model download, server check

vllm serve Qwen/Qwen2.5-7B-Instruct --port 8000 &
MODEL=Qwen/Qwen2.5-7B-Instruct bash run_paper.sh   # every single-model experiment
bash sweep_backends.sh                             # Figures 3 and 4: one model at a time
```

### Section 7 (one A100 80 GB, about 15 hours)

```bash
cd agent
pip install -r requirements.txt
python -m pytest tests -q                          # 27 tests, no GPU

vllm serve Qwen/Qwen2.5-72B-Instruct-AWQ --served-model-name qwen72b \
    --enable-auto-tool-choice --tool-call-parser hermes \
    --max-model-len 32768 --enable-prefix-caching --port 8000 &

bash scripts/run_all.sh                            # E1-E4, resumable, writes results/ and logs/
```

`run_all.sh` checks that tool calling works, runs the tests, and after the
banking part of E1 stops if the attack never reaches the undefended arm,
since the comparison would then be empty. Re-running it skips finished work.
`python run.py --backend mock ...` runs the whole pipeline without a model;
mock numbers are marked MOCK and only check the plumbing.

---

## Design choices fixed before any run

- **Untrusted tools** (Section 7): every AgentDojo tool whose output carried
  an injection canary when all reference solutions were replayed, plus tools
  that return third-party messages, emails or files. A test re-derives the
  canary carriers and fails if one is missing.
- **Sensitive actions**: every tool that changes the environment, plus
  Slack's `get_webpage`, which logs the URL it visits. A test replays every
  reference call and fails if a state-changing call is not on the list.
- **Action rule**: a sensitive action goes through only if every item in the
  recorded read set is trusted. Reads are never blocked. Refused calls do not
  count as executed when AgentDojo grades a task.
- **Handles** are assigned by the orchestrator and never appear in a prompt.
- **Router**: a fixed taxonomy over task fields; the label is part of the
  bucket key.

---

## Layout

```
README.md            this file
reproduce.py         prints every reported number from the stored results
requirements.txt     dependencies for the checks above
compaction/
  provenire/         lattice, compaction pipeline, policy check
  harness/           one script per experiment (Sections 3, 4, 6)
  analysis/          similarity sweep, end-to-end composition (Appendix D)
  results/           reported numbers, one CSV per table or figure
  tests/             37 tests
  run_paper.sh       re-runs every single-model experiment
  sweep_backends.sh  Figures 3 and 4
agent/
  silomem_agent/     labels, memory store, guard, attacks, runner, extra tasks
  run.py             multi-session episodes on AgentDojo (resumable, parallel)
  summarize.py       tables with Wilson intervals and paired McNemar tests
  diagnose.py        where the payload went: read, stored, retrieved, tried, succeeded
  scripts/run_all.sh one-click E1-E4
  results/qwen72b/   Qwen2.5-72B transcripts and summaries
  tests/             27 tests
```

## License

MIT; see [`LICENSE`](LICENSE). AgentDojo is used as a dependency under its own
license.
