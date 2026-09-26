# aida-compliance — Operational System Monitoring

**Submission type:** Operational System Monitoring
**Author:** ZARGUI Rayen — rayen.zargui@ca-cib.com (solo)
**Archive:** the skill folder `aida-operational-monitoring/`, complete — `SKILL.md`, four
scripts, the monitor template, the Table 5 reference. It is one of the seven skills of the
`aida-compliance` GitHub Copilot plugin and runs inside it: it reads the records of
`aida-record-keeping`, writes its alerts into the same journal, and relies on `aida-core`
for the scan — declared in `depends_on`, listed file by file in `SKILL.md` ›
Prerequisites. Python standard library only; nothing to install.

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
**generates the monitor that runs** — code, justified thresholds with noise guards,
schedule, and alerts that explain themselves and are **sent to the overseers**, not only
written to a journal.

## What does it do, and how?

1. **Scale to the risk (template §5)** — rates stakes, complexity and exposure from the
   code's evidence and the risk owner's answers, then applies each addition beyond Table 5,
   or leaves it out, with the reason.
2. **Plan** — reads the record-keeping audit: for each of the 13 metrics of Table 5, the
   record it needs and whether the project keeps it. The metrics selected are the ones
   that can be measured; the rest are gaps to close in record keeping first.
3. **Generate** — writes `aida/monitoring/ops_monitor.py`, its configuration (each threshold
   with its justification and noise guard, the alert channels) and its schedule.
4. **Check resilience** — every outbound call (model endpoint, vector store, HTTP) is read
   on the syntax tree for a timeout, a retry and a fallback.
5. **Write** the operational half of section 3.8 of the AIDA document — describing the
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
| Thresholds tailored and validated | every rule carries the Table 5 default, its source and **why that value**; a tailored value replaces it with its own justification; validation kept as an open point | `generate_ops_monitor.JUSTIFICATION`, `build_config` | `test_thresholds_are_the_guideline_numbers`, `test_every_threshold_carries_its_justification_and_a_noise_guard` |
| No alert on an isolated event | per-metric noise guard, with its reason: 20 requests behind a rate or an average (fewer is insufficient data — not an alert, not a pass), three consecutive samples for a resource peak, a minimum hourly volume for a spike, a count of consecutive breached periods (1 by default: the period aggregate is the guideline's smoothing) | `generate_ops_monitor.NOISE_GUARDS`, `ops_monitor.guard_holds`, `streak` | `test_too_few_requests_are_insufficient_data_not_an_alert`, `test_an_isolated_resource_peak_is_held_and_a_sustained_one_alerts`, `test_consecutive_breaches_are_counted_across_periods` |
| Frequency from inference frequency (`MON-FREQUENCY`) | crontab line and scheduled GitLab CI job at that cadence | `generate_ops_monitor.schedules` | `test_everything_the_monitor_needs_is_generated` |
| No false comfort | a metric whose record is missing is a **coverage gap, never a pass** | `ops_monitor.run` | `test_a_metric_without_its_record_is_a_gap_not_a_pass` |
| Alert, explained, recorded and **sent** | value, threshold and its justification, **trend** over the previous periods, **probable cause** read from the records (most frequent errors, the dependency timing out, the largest consumers…), next steps, owner — appended to the hash-chained journal and **sent by e-mail (SMTP) or webhook** to the configured overseers; exit code 1 turns the job red | `ops_monitor.evaluate`, `trend`, `probable_cause`, `dispatch`, `send_email`, `post_webhook` | `test_an_alert_carries_its_trend_and_probable_cause`, `test_an_alert_is_mailed_to_the_overseers`, `test_a_webhook_must_be_https`, `test_error_timeout_and_latency_breaches_are_alerted_and_journaled` |
| Dependency resilience | timeout, retry, fallback per outbound call; a call with no timeout is blocking | `check_resilience.check` | `test_a_call_without_a_timeout_is_blocking`, `test_a_call_inside_a_try_counts_as_having_a_fallback` |
| Section 3.8 (operational) | metrics and why, thresholds, tools and processes, logging, alert procedure | `write_section.build` | `test_operational_section_carries_no_quality_metric` |
| The whole table, once | the 13 metrics of Table 5, each planned once, for the system the scan classified | `plan_operational.build_plan`, `system_of` | `test_every_metric_of_the_guideline_table_is_planned_once`, `test_the_plan_names_the_system_it_was_written_for` |
| **§5 Scaling to the risk** | stakes, complexity and exposure rated with evidence — the same ratings as the GenAI monitoring skill, so both halves of 3.8 rest on one assessment; an unrated dimension stays "to confirm" and keeps its additions. Stakes send alerts from medium severity; complexity adds latency, errors and timeouts per dependency or tool; exposure adds requests and tokens per consumer (per-tenant cost) and the unrecognised-address check of Table 5. Table 5 is never scaled away; every decision is written with its reason | `scale_to_risk.profile`, `scale`, `ops_monitor.per_dependency`, `per_actor`, `unknown_ip_access` | `test_the_two_monitoring_skills_rate_the_risk_identically`, `test_a_low_risk_system_gets_table_5_and_nothing_more`, `test_the_additions_turned_on_appear_in_the_period` |

Scripts named here are in the archive. The tests are in the plugin's `tests/` folder: 695 tests, all passing on this exact skill folder. `make_submission.py --skill` re-runs them with the archived copy in place before declaring the zip ready.

A real alert, produced by the generated monitor on a simulated week in which one call in
five failed, after a previous period already in breach — and mailed to the risk owner:

```json
{
  "severity": "high",
  "metric": "error_rate",
  "value": 21.429,
  "threshold": ">= 2% per week",
  "justification": "past one call in fifty, failures are no longer incidental",
  "explanation": "error_rate: 21.4286 % >= 2 (>= 2% per week, MON-OPERATIONAL-MINIMUM), over 56 records of the period, 2 period(s) in breach.",
  "trend": {"previous": [1.8, 6.0], "direction": "rising"},
  "probable_cause": "most frequent errors: ReadTimeout: ticket-api (6), KeyError: 'status' (6)",
  "dispatched": ["email: sent"]
}
```

The same monitor reports, honestly, what it cannot compute: a metric with no record is a
coverage gap, and one with too few records is insufficient data — never a pass.

## How can we use your skill?

**Run it end to end, without installing anything** — the complete `aida-compliance` plugin is submitted as `aida-compliance-full-v2`. From its folder, `python run_all.py --demo` rebuilds the example project and writes its AIDA document, audit and runnable monitors in one command.

**In GitHub Copilot**: install the plugin from the hub marketplace and ask *"Set up
operational monitoring for this system"* or *"What can we monitor in production today?"*.
The skill asks, in one block, how often the system is called, its AI Act risk category,
internal or external users, and where alerts should go, then runs:

```bash
python skills/aida-core/scripts/scan_codebase.py . --out aida/context.json
python skills/aida-record-keeping/scripts/audit_records.py . --out aida/records_audit.json
python skills/aida-operational-monitoring/scripts/scale_to_risk.py --context aida/context.json --risk-category high --users external --inference-frequency daily
python skills/aida-operational-monitoring/scripts/plan_operational.py --context aida/context.json --records aida/records_audit.json --inference-frequency daily --risk-category high --users external
python skills/aida-operational-monitoring/scripts/generate_ops_monitor.py --plan aida/operational_plan.json --out aida/monitoring --token-quota 2000000 --email-to overseer@example.com --smtp-host smtp.internal
python skills/aida-operational-monitoring/scripts/check_resilience.py . --out aida/resilience.json
python skills/aida-operational-monitoring/scripts/write_section.py --plan aida/operational_plan.json --resilience aida/resilience.json --monitor aida/monitoring/ops_monitor_config.json --values aida/values.json --map aida/placeholders.json
```

Then the monitor itself, by hand or from `aida/monitoring/schedule/`:

```bash
python aida/monitoring/ops_monitor.py --config aida/monitoring/ops_monitor_config.json
```

## Repo structure

```
aida-operational-monitoring/              ← this archive
├── SKILL.md                              workflow, prerequisites, absolute rules, checklist
├── scripts/  plan_operational.py         metrics, feeding records, alert rules
│             generate_ops_monitor.py     plan -> runnable monitor and schedule
│             check_resilience.py         timeout / retry / fallback per call
│             scale_to_risk.py            §5: stakes, complexity, exposure; each addition decided
│             write_section.py            section 3.8, operational monitoring
├── assets/ops_monitor.py.template        the monitor that runs, explains and sends its alerts
└── references/operational_monitoring.md  Table 5 and the alert procedure
```

In the `aida-compliance` plugin it sits next to `aida-record-keeping/` (the records the
monitor reads, the journal it writes), `aida-core/` (scan, 36 rules, validator, Word
writer), `examples/` and `tests/` (`test_ops_monitor.py`, `test_monitoring.py`,
`test_runtime_integration.py` …).

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
  this task activates one `SKILL.md` (~3,400 tokens). Scripts are executed, not read.

## Does it use external access?

**The skill: no.** Its scripts read local files and write under `aida/`; no network
module, no telemetry. **The generated monitor: only to deliver alerts**, to the channels the
team configures — none by default. E-mail goes through the team's SMTP relay with the
password read from `AIDA_SMTP_PASSWORD`, never written to the config; a webhook must be
https; the payload holds metric values, the explanation and actor ids, never an input or
an output. Enforced by tests:

| Agent Skills risk | What holds | Enforced by |
|---|---|---|
| AST01 exfiltration | no network module in any skill script; the monitor sends only to configured channels, https webhooks only, credentials from the environment | `test_no_script_imports_a_network_module`, `test_a_webhook_must_be_https`, `test_an_alert_is_mailed_to_the_overseers` |
| AST02 supply chain | standard library only; every declared dependency pinned | `test_every_declared_dependency_is_pinned_to_an_exact_version` |
| AST03 permissions | `allowed-tools` limited to `shell(python:*)` for this skill | `test_allowed_tools_is_scoped_to_named_commands` |
| AST05 unsafe execution | no `eval` / `exec` / `pickle`; JSON only; no shell subprocess | `test_no_script_evaluates_or_unpickles_input`, `test_subprocess_is_never_given_a_shell` |

## Limitations / known issues

- **Metrics need records.** The monitor computes what the journal holds; host metrics
  (CPU, memory) arrive through `resource_usage` records the platform must write.
- **Thresholds need validating.** The defaults are the guideline's; tailoring them to the
  system and having them validated before the end of the build phase stays with the team.
- **Delivery needs a channel.** The monitor mails or posts each alert; the relay, the
  recipients and the webhook are the team's to give. Without one, alerts stay in the
  journal and the scheduled job turns red — and section 3.8 says so as an open point.
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
