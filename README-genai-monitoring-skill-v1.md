# aida-compliance — GenAI Monitoring

**Submission type:** GenAI Monitoring
**Author:** ZARGUI Rayen — rayen.zargui@ca-cib.com (solo)
**Archive:** the skill folder `aida-genai-monitoring/`, complete — `SKILL.md`, four scripts,
the monitor template, the metric reference. It is one of the seven skills of the
`aida-compliance` GitHub Copilot plugin and runs inside it: it relies on `aida-core` for the
scan and on the journal `aida-record-keeping` defines, both declared in `depends_on` and
listed file by file in `SKILL.md` › Prerequisites. Python standard library only; nothing to
install.

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

1. **Scope** — decides the family and the variant from evidence, and rules out what is
   absent. A system that is not GenAI gets no monitor.
2. **Plan** — the family's metrics, the **latest** build-phase baseline, thresholds derived
   from it, the window and the frequency.
3. **Generate** — `aida/monitoring/genai_monitor.py`, its configuration, alert rules and
   schedule.
4. **Write** section 3.8 (AI Model Monitoring) of the AIDA document — the monitor that runs,
   not an intention.

### Coverage of the guideline

| Family | What the skill builds | Produced by | Proven by |
|---|---|---|---|
| **Scope** | variant **with ground truth** (records audit or scan finds ground truth or feedback) or **without**; absent families ruled out with their evidence (*"rag - scan: 0 retrieval signal(s), 0 vector store(s)"*); no monitor for a system not established as GenAI | `plan_monitoring.scope` | `test_scope_is_decided_with_evidence`, `test_no_monitor_is_generated_for_a_system_that_is_not_genai` |
| **Metrics** | task completion, tool correctness, latency, tokens from the inference records; token-F1 correctness and retrieval recall / precision / MRR / NDCG against ground truth; faithfulness, relevancy, reasoning through `register_judge` (DeepEval, RAGAS, in-house) | `genai_monitor.compute` | `test_a_clean_period_raises_nothing`, `test_retrieval_metrics_from_ids`, `test_a_judge_metric_without_a_judge_is_a_gap_and_with_one_is_computed` |
| **Prompt extra** | the prompt table in full: the three judged metrics plus answer correctness and the usefulness score (Likert 1-5) from user feedback | `plan_monitoring.FAMILY_METRICS`, `genai_monitor.compute` | `test_a_prompt_system_gets_the_two_human_feedback_metrics_of_its_table`, `test_usefulness_is_the_users_likert_score_scaled` |
| **RAG extra** | retrieval ranks computed from retrieved and relevant ids; knowledge-base and question-set drift planned | `genai_monitor.ranking`, `plan_monitoring.DRIFT_PLAN` | `test_a_rag_system_gets_retrieval_metrics_too`, `test_a_rag_system_gets_no_tool_metrics` |
| **Baseline** | the most recent evaluation summary the project holds; none means no threshold and a blocking finding, never a default | `plan_monitoring.find_baseline` | `test_the_baseline_is_the_most_recent_evaluation`, `test_no_baseline_at_all_is_a_blocking_finding` |
| **Threshold** | `baseline × (1 − drop)` from the family's rule; latency and tokens alert on a rise; a looser threshold is blocked by the validator | `plan_monitoring.metric_entry` | `test_threshold_is_derived_from_the_measured_score`, `test_latency_degrades_upwards_even_without_a_baseline` |
| **Collection** | period from `MON-FREQUENCY`; crontab line and scheduled GitLab CI job; a period below the evaluation minimum is **insufficient data**, not a pass | `generate_monitor.schedules`, `genai_monitor.run` | `test_schedule_follows_mon_frequency`, `test_too_few_inferences_is_insufficient_data_not_a_pass` |
| **Window** | one period past the drop alerts, as the rule says; consecutive breaches configurable and tracked | `plan_monitoring.window`, `genai_monitor.consecutive` | `test_consecutive_breaches_are_honoured` |
| **Alert + explain** | value, baseline, threshold, relative change, rule, negative feedback by cause, requests to review, the four next steps — appended to the hash-chained journal | `genai_monitor.quality_alert` | `test_a_quality_drop_is_alerted_explained_and_journaled` |
| **Safety** | personal data in any answer is a critical alert, no baseline needed | `genai_monitor.safety_alert` | `test_pii_in_an_answer_is_a_critical_alert` |
| **Ground truth** | user feedback (`up` / `down` with its cause) and corrections joined to each inference by `request_id` | `genai_monitor.ground_truth` | `test_ground_truth_and_feedback_are_joined_by_request_id` |
| **Drift** | query mix against a reference (chi-square, p < 0.05); tool-set drift between two git revisions (`MON-DRIFT-TOOLSET`) | `genai_monitor.chi_square_p`, `detect_tool_drift.diff` | `test_query_drift_is_tested_against_the_reference`, `test_an_added_tool_is_a_critical_alert` |
| Section 3.8 | written from the plan, naming the monitor that runs; a metric that degrades upwards is alerted above its threshold | `write_section.build` | `test_section_3_8_describes_what_runs_not_what_is_intended`, `test_a_metric_that_degrades_upwards_is_alerted_above_its_threshold` |

Scripts named here are in the archive. The tests are in the plugin's `tests/` folder: 670 tests, all passing on this exact skill folder. `make_submission.py --skill` re-runs them with the archived copy in place before declaring the zip ready.

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

**Run it end to end, without installing anything** — the complete `aida-compliance` plugin is submitted as `aida-compliance-full-v2`. From its folder, `python run_all.py --demo` rebuilds the example project and writes its AIDA document, audit and runnable monitors in one command.

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

Then the monitor itself, by hand or from the generated schedule:

```bash
python aida/monitoring/genai_monitor.py --config aida/monitoring/genai_monitor_config.json
```

The records it reads come from the application through the generated record module —
`@records.logged_inference(model_version=...)` captures input, output, latency, tokens,
tool calls and outcome on every call. To see a result first, open
`examples/ticket_agent/technical_documentation.docx` in the plugin: section 3.8 and the
§3.4 baseline-vs-threshold chart come from this skill.

## Repo structure

```
aida-genai-monitoring/                  ← this archive
├── SKILL.md                            6-phase workflow, prerequisites, absolute rules, checklist
├── scripts/  plan_monitoring.py        scope, metrics, baselines, thresholds, window
│             generate_monitor.py       plan -> runnable monitor, rules, schedule
│             detect_tool_drift.py      tool-set drift between two git revisions
│             write_section.py          section 3.8, model monitoring
├── assets/genai_monitor.py.template    the monitor that runs
└── references/genai_monitoring.md      metric tables per family, drift, feedback
```

In the `aida-compliance` plugin it sits next to `aida-core/` (scan, 36 rules, validator,
Word writer), `aida-record-keeping/` (the journal the monitor reads and writes),
`aida-operational-monitoring/` (the operational half of 3.8, same journal), `examples/` and
`tests/` (`test_genai_monitor.py`, `test_monitoring.py`, `test_runtime_integration.py` …).

## Tested on a real project?

**Yes — three real projects of the Plugin-Skill-Hub, written by other people**, fixtures,
and an example project run through a simulated production day.

| Project | What the skill decided |
|---|---|
| promptopt (prompt optimiser) | prompt family, **no ground truth**; agentic and analytics ruled out with evidence; faithfulness, relevancy and correctness need a judge, answer correctness and usefulness need user feedback — and the skill says which |
| AgentGrade engine | family not established from the code: **no monitor generated**, the reason stated |
| hub-skill-installer | not an AI system: **no monitor generated** |
| ticket agent (example) | agentic L2; baseline from the most recent of two evaluation runs (0.86); in a simulated day, 32 of 40 tasks completed (0.80, above the 0.774 threshold): **no alert, correctly** |

**Running on real projects found three faults, all fixed and pinned by tests**: a monitor
was generated for systems that are not GenAI; with two evaluation runs the older one was
taken as the baseline; and latency was treated as higher-is-better when no baseline had
been measured. In the simulated day the generated record module, oversight controls and
both monitors shared one journal that verified **INTACT**
(`test_records_oversight_and_monitors_share_one_verifiable_journal`).

## Models / tools used

- **No model is called by the skill.** Scope, thresholds, generation and the monitor itself
  are deterministic Python. The only model in the loop is the judge the team registers for
  judge metrics, and the monitor states which metrics need one.
- **Standard library only**, including the chi-square p-value (regularised incomplete gamma)
  and the ranking metrics, so the monitor runs anywhere Python 3.10 does.
- **Context footprint:** the seven skill descriptions (~1,200 tokens) are always loaded; this
  task activates one `SKILL.md` (~2,800 tokens). Scripts are executed, not read.

## Does it use external access?

**No.** The skill and the generated monitor read local files (the project, the record
journal) and write under `aida/`. No network call, no telemetry. Enforced by tests:

| Agent Skills risk | What holds | Enforced by |
|---|---|---|
| AST01 exfiltration | no network module in any script; writes only under `aida/` | `test_no_script_imports_a_network_module`, `test_drift_detection_never_touches_the_working_tree` |
| AST02 supply chain | standard library only; every declared dependency pinned | `test_every_declared_dependency_is_pinned_to_an_exact_version` |
| AST03 permissions | `allowed-tools` is `shell(python:*)` alone; tool-set drift reads git objects through fixed read-only commands (`ls-tree`, `show`), never a shell | `test_allowed_tools_is_scoped_to_named_commands`, `test_the_agent_is_never_granted_git`, `test_scripts_only_read_git` |
| AST05 unsafe execution | no `eval` / `exec` / `pickle`; JSON only; no shell subprocess | `test_no_script_evaluates_or_unpickles_input`, `test_subprocess_is_never_given_a_shell` |

## Limitations / known issues

- **Judge metrics need a judge.** Faithfulness, relevancy, reasoning quality and toxicity
  cannot be computed from records alone. The monitor exposes `register_judge` and reports an
  unjudged metric as a coverage gap. It never guesses a score.
- **The records must carry the fields.** Task completion needs an `outcome`, retrieval
  metrics need `retrieved_ids` and ground-truth `relevant_ids`; the generated record module
  writes the first automatically, the retrieval ids are the application's to pass.
- **Query drift uses task types when records carry them**, and question-length bands
  otherwise. Clustering on embeddings, as the guideline suggests, is left to the team.
- **Static scope detection.** A family configured only at deployment time may be missed.
  The scope is reported with its evidence so a human can overrule it.

## Anything else worth knowing?

- **One journal for everything.** The monitor reads the records written by the
  record-keeping deliverable and writes its alerts into the same hash-chained file, so an
  auditor verifies inferences, feedback, interventions and alerts with one command.
- **Section 3.8 of the AIDA document is generated from the same plan**, so what the
  document says and what runs cannot diverge.
- **Stricter, never looser.** The validator blocks a threshold looser than the guideline;
  a stricter one is accepted with a warning asking for its justification.

---

*Built against the AI Factory guidelines: Monitoring and Human Oversight, Evaluation of
GenAI Systems, Record Keeping. Follows `templates/plugin-template` and
`templates/skill-template` of the Plugin-Skill-Hub.*
