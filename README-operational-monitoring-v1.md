# aida-compliance — Operational System Monitoring

**Submission type:** Operational System Monitoring
**Author:** ZARGUI Rayen — rayen.zargui@ca-cib.com (solo)
**Archive:** a GitHub Copilot plugin. The skill for this track is
`aida-operational-monitoring`; it reads the records of `aida-record-keeping`, writes its
alerts into the same journal, and writes the operational half of section 3.8 of the AIDA
technical documentation. Python standard library only.

---

## What problem does it solve?

The monitoring guideline asks for at least one metric from its operational table
(`MON-OPERATIONAL-MINIMUM`, Table 5) — latency, uptime, error and timeout rates, crashes,
resource use, unauthorised access, usage trends, token consumption — with thresholds
tailored to the system, validated **before the end of the build phase** (`MON-TIMING`), and
computed at a frequency set by the inference frequency (`MON-FREQUENCY`). Picking metrics
from a table is easy. Picking ones the system can **measure** is the part that gets
skipped: a project that never records a timeout cannot report a timeout rate.

This skill ties each metric to the record that feeds it, reports which ones the project
can compute today, checks every outbound call for timeout, retry and fallback, and
**generates the monitor that runs** — code, thresholds, schedule, alerts into the journal.

## What does it do, and how?

1. **Plan** — reads the record-keeping audit: for each of the 13 metrics of Table 5, the
   record it needs and whether the project keeps it. The metrics selected are the ones
   that can be measured; the rest are gaps to close in record keeping first.
2. **Generate** — writes `aida/monitoring/ops_monitor.py`, its configuration and its
   schedule.
3. **Check resilience** — every outbound call (model endpoint, vector store, HTTP) is read
   on the syntax tree for a timeout, a retry and a fallback.
4. **Write** the operational half of section 3.8 of the AIDA document — describing the
   monitor that runs, not an intention.

### Coverage of the guideline

| Requirement | How it is met | Produced by | Proven by |
|---|---|---|---|
| At least one Table 5 metric, measurable | each metric tied to its feeding record; computability from the record audit, never assumed | `plan_operational.build_plan` | `test_metrics_are_reported_with_the_record_that_feeds_them`, `test_without_a_record_audit_computability_is_unknown_not_assumed` |
| Implemented before the end of the build phase (`MON-TIMING`) | a monitor that runs, generated from the plan | `generate_ops_monitor.build_config`, `ops_monitor.py` | `test_everything_the_monitor_needs_is_generated` |
| Reliability metrics | latency ≥ 3 s, processing ≥ 3 s, output ≥ 2000 tokens, uptime < 99 %, timeouts ≥ 5 %, errors ≥ 2 %, one crash in 24 h, resources ≥ 90 % | `ops_monitor.reliability` | `test_error_timeout_and_latency_breaches_are_alerted_and_journaled`, `test_one_crash_in_24_hours_is_critical` |
| Security metric | ≥ 3 failed logins by one actor within one minute | `ops_monitor.failed_logins_per_minute` | `test_three_failed_logins_within_a_minute_is_an_alert` |
| Usage metrics | request spike +200 % in an hour; active users −30 %; session length < 20 % or > 3× the previous periods | `ops_monitor.usage`, `breached` | `test_a_request_spike_and_a_token_quota_breach`, `test_active_users_falling_30_percent_against_history` |
| FinOps metric | daily tokens against a quota the team sets (`--token-quota`) | `ops_monitor.usage` | `test_a_request_spike_and_a_token_quota_breach` |
| Thresholds tailored and validated | guideline defaults in the config, tailoring with its justification, validation kept as an open point | `generate_ops_monitor.build_config` | `test_thresholds_are_the_guideline_numbers` |
| Frequency from inference frequency (`MON-FREQUENCY`) | crontab line and scheduled GitLab CI job at that cadence | `generate_ops_monitor.schedules` | `test_everything_the_monitor_needs_is_generated` |
| No false comfort | a metric whose record is missing is a **coverage gap, never a pass** | `ops_monitor.run` | `test_a_metric_without_its_record_is_a_gap_not_a_pass` |
| Alert, explained and recorded | value, threshold, rule, explanation, next steps, owner — appended to the hash-chained journal; exit code 1 turns the job red | `ops_monitor.run` | `test_error_timeout_and_latency_breaches_are_alerted_and_journaled`, `test_the_cli_exits_1_on_an_alert` |
| Dependency resilience | timeout, retry, fallback per outbound call; a call with no timeout is blocking | `check_resilience.check` | `test_a_call_without_a_timeout_is_blocking`, `test_a_call_inside_a_try_counts_as_having_a_fallback` |
| Section 3.8 (operational) | metrics and why, thresholds, tools and processes, logging, alert procedure | `write_section.build` | `test_operational_section_carries_no_quality_metric` |

A real alert, produced by the generated monitor on the example project during a simulated
day in which one call in five failed:

```json
{
  "kind": "operational",
  "severity": "high",
  "rule": "MON-OPERATIONAL-MINIMUM",
  "metric": "error_rate",
  "value": 40.0,
  "threshold": ">= 2% per week",
  "explanation": "error_rate: 40 % >= 2 (>= 2% per week, MON-OPERATIONAL-MINIMUM), over 56 records of the period.",
  "next_steps": ["Investigate: new usage pattern, dependency outage, deployment, capacity",
                 "Justify whether corrective measures are taken, and if not, why",
                 "Correct: scale, fix, roll back, tighten access",
                 "Record the investigation and the measures in the AI System Journal (DOC-JOURNAL)"],
  "journal_seq": 57
}
```

The same run raised a second alert — three failed logins from one address within a minute —
and reported, honestly, three metrics it could not compute: processing time and resource
use (no record kept) and token usage (no quota set).

## How can we use your skill?

**In GitHub Copilot**: install the plugin from the hub marketplace and ask *"Set up
operational monitoring for this system"* or *"What can we monitor in production today?"*.
The skill asks how often the system is called, then runs:

```bash
python skills/aida-core/scripts/scan_codebase.py . --out aida/context.json
python skills/aida-record-keeping/scripts/audit_records.py . --out aida/records_audit.json
python skills/aida-operational-monitoring/scripts/plan_operational.py --context aida/context.json --records aida/records_audit.json --inference-frequency daily
python skills/aida-operational-monitoring/scripts/generate_ops_monitor.py --plan aida/operational_plan.json --out aida/monitoring --token-quota 2000000
python skills/aida-operational-monitoring/scripts/check_resilience.py . --out aida/resilience.json
python skills/aida-operational-monitoring/scripts/write_section.py --plan aida/operational_plan.json --resilience aida/resilience.json --values aida/values.json --map aida/placeholders.json
```

Then the monitor itself, by hand or from `aida/monitoring/schedule/`:

```bash
python aida/monitoring/ops_monitor.py --config aida/monitoring/ops_monitor_config.json
```

## Repo structure

```
aida-compliance/
├── plugin.json  plugin.yaml  README.md  run_all.py
├── skills/
│   ├── aida-operational-monitoring/    ← this submission
│   │   ├── SKILL.md                    workflow, absolute rules, quality checklist
│   │   ├── scripts/  plan_operational.py      metrics, feeding records, alert rules
│   │   │             generate_ops_monitor.py  plan -> runnable monitor and schedule
│   │   │             check_resilience.py      timeout / retry / fallback per call
│   │   │             write_section.py         section 3.8, operational monitoring
│   │   ├── assets/ops_monitor.py.template     the monitor that runs
│   │   └── references/operational_monitoring.md   Table 5 and the alert procedure
│   ├── aida-record-keeping/            the records the monitor reads, the journal it writes
│   └── aida-core/                      scan, 36 rules, validator, Word writer
├── examples/ticket_agent/              a generated AIDA document to open directly
└── tests/  test_ops_monitor.py  test_monitoring.py  test_runtime_integration.py …
```

## Tested on a real project?

**Yes — three real projects of the Plugin-Skill-Hub**, fixtures, and an example project run
through a simulated production day.

| Project | Metrics computable today | Outbound calls without timeout | What the skill reports |
|---|---|---|---|
| ticket agent (example) | 0 of 13 before record keeping; 10 once the generated records are wired in | 1 of 2 (`agent.invoke`) | the gap, the call site, and the monitor ready to run |
| promptopt | 0 of 13 | 1 of 1 | the model call has no timeout: latency cannot be bounded |
| AgentGrade engine | 0 of 13 | none found | every metric waits on a record |
| hub-skill-installer | 0 of 13 | 0 of 1 | the one HTTP call is bounded |

None of these projects records what Table 5 needs — and the skill says so rather than
select metrics it cannot measure. Once the generated record module is wired in, the same
monitor computes them: in the simulated day, **10 of the 13 metrics were measured**, two
alerts raised and journaled, and the journal verified intact
(`test_records_oversight_and_monitors_share_one_verifiable_journal`).

## Models / tools used

- **No model is called.** Metrics, thresholds and alerts are deterministic Python.
- **Standard library only**, in the generated monitor too: it runs next to the application.
- **Context footprint:** the seven skill descriptions (~1,200 tokens) are always loaded;
  this task activates one `SKILL.md` (~2,100 tokens). Scripts are executed, not read.

## Does it use external access?

**No.** The skill and the monitor read local files and write under `aida/` and to the
local journal. No network call, no telemetry. Enforced by tests:

| Agent Skills risk | What holds | Enforced by |
|---|---|---|
| AST01 exfiltration | no network module in any script; writes only under `aida/` | `test_no_script_imports_a_network_module`, `test_project_source_is_never_touched` |
| AST02 supply chain | standard library only; every declared dependency pinned | `test_every_declared_dependency_is_pinned_to_an_exact_version` |
| AST03 permissions | `allowed-tools` limited to `shell(python:*)` for this skill | `test_allowed_tools_is_scoped_to_named_commands` |
| AST05 unsafe execution | no `eval` / `exec` / `pickle`; JSON only; no shell subprocess | `test_no_script_evaluates_or_unpickles_input`, `test_subprocess_is_never_given_a_shell` |

## Limitations / known issues

- **Metrics need records.** The monitor computes what the journal holds; host metrics
  (CPU, memory) arrive through `resource_usage` records the platform must write.
- **Thresholds need validating.** The defaults are the guideline's; tailoring them to the
  system and having them validated before the end of the build phase stays with the team.
- **Delivery channel.** Alerts are journaled and turn the scheduled job red; wiring the
  job's failure to mail or chat is the platform's.
- **Static resilience check.** A timeout set through configuration the code does not show
  is reported as missing, to confirm.

## Anything else worth knowing?

- **Model quality is not this section.** Faithfulness and task completion belong to the
  GenAI monitor, generated by the sibling skill from the same plan style and the same
  journal; the operational section carries no quality metric
  (`test_operational_section_carries_no_quality_metric`).
- **One journal.** Inferences, oversight interventions and both monitors' alerts are
  chained together; one command verifies them.
- **Section 3.8 of the AIDA document** is written from the same plan and names the monitor
  that runs.

---

*Built against the AI Factory Monitoring and Human Oversight guideline (Table 5) and the
Record Keeping guideline. Follows `templates/plugin-template` and `templates/skill-template`.*
