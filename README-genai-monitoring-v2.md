# aida-compliance — GenAI Monitoring

**Submission type:** GenAI Monitoring
**Author:** ZARGUI Rayen — rayen.zargui@ca-cib.com (solo)
**Archive:** a GitHub Copilot plugin. The skill for this track is `aida-genai-monitoring`; it
relies on `aida-core` for the scan and on `aida-record-keeping` for the journal it reads.
Python standard library only; nothing to install.

---

## What problem does it solve?

The monitoring guideline asks for GenAI quality monitoring that is **in place before the end
of the build phase** (`MON-TIMING`): the metrics of the build-phase evaluation, recomputed on
production traffic at a frequency set by the inference frequency, with an alert on a 20 %
(prompt) or 10 % (RAG, agentic) relative drop, and every substantial alert investigated and
recorded. Teams usually stop at a document that promises this. Nothing runs, so nothing
alerts.

This skill reads the project, decides what must be monitored and why, and **generates the
monitor that does it**: code, configuration, alert rules and schedule, wired to the same
tamper-evident journal the record-keeping deliverable uses.

## What does it do, and how?

| Guideline family | What the skill builds |
|---|---|
| **Scope** | Decides the variant **with ground truth** (the records audit or the scan finds ground truth or user feedback collected) or **without**, and rules out absent families with their evidence: *"rag - scan: 0 retrieval signal(s), 0 vector store(s)"*. A system that is not GenAI gets no monitor. |
| **Metrics** | Per family: task completion, tool correctness, latency and tokens from the inference records; correctness (token-F1) and retrieval recall, precision, MRR, NDCG against ground truth; answer correctness from user feedback; PII in any answer. Faithfulness, relevancy and reasoning quality go through `register_judge` (DeepEval, RAGAS, in-house). A metric with no data and no judge is a **coverage gap, never a pass**. |
| **Baseline** | Read from the project's evaluation output (`eval/outputs/*/summary.json`). No baseline means no threshold and a blocking finding, never a default number. |
| **Threshold** | `baseline × (1 − drop)`, with the drop taken from the family's rule (`MON-THRESHOLD-PROMPT` 20 %, `-RAG` / `-AGENTIC` 10 %). Latency and tokens alert on a rise. The validator blocks any looser threshold. |
| **Collection** | Period from `MON-FREQUENCY` (daily inference to weekly monitoring, and so on). A generated `crontab` line and a scheduled GitLab CI job run the monitor; exit code 1 turns the job red. |
| **Window** | One period past the drop triggers the alert, as the rule says; more periods would loosen it and require a justification. A period with fewer inferences than the evaluation minimum (`EVAL-*`: 20 agentic, 50 RAG) is **insufficient data**, not a pass. Consecutive breaches are tracked in a history file. |
| **Alert + explain** | Each alert carries the value, baseline, threshold, relative change, rule, periods in breach, negative feedback by cause (generation, retrieval, tools), the requests to review and the guideline's four next steps. It is appended to the **hash-chained record journal**, so `verify_chain.py` audits it like any other record (`DOC-JOURNAL`). |
| **Drift** | Query mix tested against a reference distribution (chi-square, p < 0.05). Tool-set drift between two git revisions (`MON-DRIFT-TOOLSET`): any tool or parameter added or removed is critical. |
| **Ground truth** | User feedback (`up` / `down` plus the cause) and corrections are joined to each inference by `request_id`. |

A real alert, produced by the generated monitor on an agentic fixture evaluated at 0.86:

```json
{
  "kind": "quality_drop",
  "severity": "high",
  "rule": "MON-THRESHOLD-AGENTIC",
  "metric": "task_completion",
  "value": 0.6,
  "baseline": 0.86,
  "alert_threshold": 0.774,
  "relative_change_pct": -30.2,
  "consecutive_breaches": 1,
  "samples": 30,
  "explanation": "task_completion is 0.600 against a build-phase baseline of 0.86 (-30.2%), past the 10% drop of MON-THRESHOLD-AGENTIC for 1 period(s).",
  "requests_to_review": ["r18", "r19", "r20", "r21", "r22"]
}
```

## How can we use your skill?

**In GitHub Copilot**: install the plugin from the hub marketplace and ask *"Set up GenAI
monitoring for this project"* or *"What should we monitor on this RAG in production?"*. The
skill asks one question (how often the system is called), then runs:

```bash
python skills/aida-core/scripts/scan_codebase.py . --out aida/context.json
python skills/aida-record-keeping/scripts/audit_records.py . --out aida/records_audit.json
python skills/aida-genai-monitoring/scripts/plan_monitoring.py --context aida/context.json --project . --inference-frequency daily
python skills/aida-genai-monitoring/scripts/generate_monitor.py --plan aida/monitoring_plan.json --out aida/monitoring
python skills/aida-genai-monitoring/scripts/write_section.py --plan aida/monitoring_plan.json --values aida/values.json --map aida/placeholders.json
```

Then run the monitor itself, by hand or from the generated schedule:

```bash
python aida/monitoring/genai_monitor.py --config aida/monitoring/genai_monitor_config.json
```

Output, in `<project>/aida/monitoring/`:

| File | Content |
|---|---|
| `genai_monitor.py` | the monitor: records to metrics, thresholds, drift, explained alerts into the journal |
| `genai_monitor_config.json` | scope, variant, ruled-out families, window, metrics with baseline, limit, direction and method |
| `alert_rules_quality.json` | one machine-readable rule per metric, plus PII and query drift |
| `schedule/crontab.txt`, `schedule/gitlab-ci.yml` | the run at the `MON-FREQUENCY` cadence |
| `monitoring_history.jsonl` | one line per period, which is what the consecutive-breach window reads |

Section 3.8 (AI Model Monitoring) of the AIDA technical documentation is written from the
same plan and describes the implemented monitor, not an intention.

## Repo structure

```
aida-compliance/
├── plugin.json  plugin.yaml  README.md  run_all.py
├── skills/
│   ├── aida-genai-monitoring/        ← this submission
│   │   ├── SKILL.md                  6-phase workflow, absolute rules, quality checklist
│   │   ├── scripts/  plan_monitoring.py   scope, metrics, baselines, thresholds, window
│   │   │             generate_monitor.py  plan -> runnable monitor, rules, schedule
│   │   │             detect_tool_drift.py tool-set drift between two git revisions
│   │   │             write_section.py     section 3.8, model monitoring
│   │   ├── assets/genai_monitor.py.template   the monitor that runs
│   │   └── references/genai_monitoring.md     metric tables per family, drift, feedback
│   ├── aida-core/                    scan, classification, 36 rules, validator, docx writer
│   ├── aida-record-keeping/          the hash-chained journal the monitor reads and writes
│   └── … (technical documentation, operational monitoring, human oversight, audit)
└── tests/  test_genai_monitor.py  test_monitoring.py  …   613 tests in total
```

## Tested on a real project?

**Yes — three real projects of the Plugin-Skill-Hub, written by other people**, plus fixtures:

| Project | What the skill decided |
|---|---|
| promptopt (prompt optimiser) | prompt family, **no ground truth**; agentic and analytics ruled out with evidence; weekly window; faithfulness, relevancy and correctness need a judge, and the skill says so |
| AgentGrade engine | family not established from the code: **no monitor generated**, and the reason stated |
| hub-skill-installer | not an AI system: **no monitor generated** |

The last two are the negative controls that matter. Monitoring a system that is not GenAI
would be inventing one, and the first version of this generator did exactly that on these
two projects; the fault is fixed and pinned by a test.

The generated monitor is tested end to end (16 tests): a clean period raises nothing; a
drop from 0.9 to 0.6 raises an explained alert whose journal still verifies; 5 inferences
are insufficient data; two consecutive breaches are required when configured; PII in an
answer is critical; feedback is joined by `request_id` (60 % positive); a judge metric is a
gap until a judge is registered and alerts after; a shifted query mix fails the chi-square
test; the CLI exits 1 on an alert.

## Models / tools used

- **No model is called by the skill.** Scope, thresholds, generation and the monitor itself
  are deterministic Python. The only model in the loop is the judge the team registers for
  judge metrics, and the monitor states which metrics need one.
- **Standard library only**, including the chi-square p-value (regularised incomplete gamma)
  and the ranking metrics, so the monitor runs anywhere Python 3.10 does.
- **Context footprint:** the seven skill descriptions (~1,200 tokens) are always loaded; this
  task activates one `SKILL.md` (~2,500 tokens). Scripts are executed, not read.

## Does it use external access?

**No.** The skill and the generated monitor read local files (the project, the record
journal) and write under `aida/`. No network call, no telemetry. Enforced by tests: no
network import, no `eval` / `exec` / `pickle`, no shell subprocess. `allowed-tools` is scoped
to `shell(python:*)`, plus `shell(git:*)` for tool-set drift.

## Limitations / known issues

- **Judge metrics need a judge.** Faithfulness, relevancy, reasoning quality and toxicity
  cannot be computed from records alone. The monitor exposes `register_judge` and reports an
  unjudged metric as a coverage gap. It never guesses a score.
- **The records must carry the fields.** Task completion needs an `outcome`, retrieval
  metrics need `retrieved_ids` and ground-truth `relevant_ids`. The generator prints the
  exact fields, and `aida-record-keeping` generates the logger that writes them.
- **Query drift uses task types when records carry them**, and question-length bands
  otherwise. Clustering on embeddings, as the guideline suggests, is left to the team.
- **Static scope detection.** A family configured only at deployment time may be missed.
  The scope is reported with its evidence so a human can overrule it.

## Anything else worth knowing?

- **One journal for everything.** The monitor reads the records written by the
  record-keeping deliverable and writes its alerts into the same hash-chained file, so an
  auditor verifies inferences, feedback and alerts with one command.
- **Section 3.8 of the AIDA document is generated from the same plan**, so what the
  document says and what runs cannot diverge.
- **Stricter, never looser.** The validator blocks a threshold looser than the guideline;
  a stricter one is accepted with a warning asking for its justification.

---

*Built against the AI Factory guidelines: Monitoring and Human Oversight, Evaluation of
GenAI Systems, Record Keeping. Follows `templates/plugin-template` and
`templates/skill-template` of the Plugin-Skill-Hub.*
