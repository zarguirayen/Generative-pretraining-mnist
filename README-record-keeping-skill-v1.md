# aida-compliance — Record Keeping

**Submission type:** Record Keeping
**Author:** ZARGUI Rayen — rayen.zargui@ca-cib.com (solo)
**Archive:** the skill folder `aida-record-keeping/`, complete — `SKILL.md`, four scripts,
the journal module template, the requirements reference. It is one of the seven skills of
the `aida-compliance` GitHub Copilot plugin and runs inside it, on the plugin's shared
engine `aida-core` (declared in `depends_on`, listed file by file in `SKILL.md` ›
Prerequisites). The journal it defines is the one the monitoring and oversight skills
write into. Python standard library only; nothing to install.

---

## What problem does it solve?

The Record Keeping guideline (EU AI Act Article 12) makes **nine records mandatory** for an
internally developed system — six for the model, two operational, one for human oversight —
and requires them **structured, timestamped, automatic at the architecture level, retained
one to ten years, in storage that cannot be altered unnoticed**. Teams usually answer with
"we log to CloudWatch": nobody can say which of the nine records exist, which fields they
carry, or prove that nothing was changed afterwards.

This skill answers the three questions an auditor asks — *which records exist, do they
carry what monitoring needs, can they be trusted* — from the code, and **generates what is
missing**: a hash-chained journal module that records at the architecture level, and a
standalone verifier an auditor runs on the file alone.

## What does it do, and how?

1. **Audit** — reads the code's syntax tree. A record has a mechanism only when **a call
   that records** — a logger, an OpenTelemetry span setter, `mlflow.log_*`, a record helper,
   a database insert — writes it, or when the platform declares it (DVC for datasets, a CI
   deployment environment). A README or a data file that mentions a record records nothing.
   It also checks the twelve fields monitoring needs, the log format, retention and
   immutability, each as a separate finding.
2. **Generate** — writes `aida/generated/aida_records.py` and prints the call site of every
   missing record. The project's source is never edited.
3. **Verify** — `verify_chain.py` checks the journal with nothing but the file.
4. **Write** section 3.7 of the AIDA document from the audit, gaps stated plainly.

### Coverage of the guideline

| Requirement | How it is met | Produced by | Proven by |
|---|---|---|---|
| `REC-MODEL-MANDATORY` — 6 model records | audited from recording calls; one helper per record (`model_dataset`, `model_performance`, `inference`, `ground_truth`) | `audit_records.audit`, `aida_records.py` | `test_all_nine_mandatory_records_are_checked`, `test_every_mandatory_record_has_a_documented_call_site` |
| `REC-OPERATIONAL-MANDATORY` — deployment, access | `deployment`, `access` helpers; a CI deployment environment counts | `audit_records.declared_mechanisms` | `test_a_recording_call_and_a_ci_environment_are_mechanisms` |
| Recommended operational records | `error_event`, `timeout_event`, `incident`, `uptime_probe`, `resource_usage` — what the operational monitor computes from | `aida_records.py` | `test_recommended_operational_records_feed_the_monitor` |
| `REC-OVERSIGHT-MANDATORY` — human review | `human_review` (accepted / overridden / escalated), alerts and follow-ups | `aida_records.py`, oversight controls | `test_overrides_and_interventions_keep_the_journal_verifiable` |
| `REC-FORMAT` — structured, timestamped, what / when / who | every entry is JSON with `event`, `at` (UTC), `actor`, `payload` | `aida_records.record` | `test_a_fresh_journal_verifies` |
| `REC-FORMAT` — automatic, at the architecture level | `@logged_inference` records every call, failures included; `RecordHandler` sends the application's logs into the journal | `aida_records.py` | `test_every_inference_is_recorded_without_a_call_site`, `test_application_logs_reach_the_chained_journal` |
| Fields monitoring needs | the twelve required fields read from real keywords and dict keys, never from message strings; OpenTelemetry attributes count | `audit_records.logged_field_names` | `test_a_percent_formatted_log_records_no_fields`, `test_span_attributes_count_as_recorded_fields` |
| `REC-RETENTION` — 10 y / 1 y, NTP time | retention policy carried in the module; timestamps UTC; gaps reported | `audit_records.audit` | `test_format_retention_and_immutability_are_three_separate_findings` |
| `REC-RETENTION` — secure, immutable storage | each entry carries the hash of the previous one; **anchors** kept on write-once storage catch a cut-off tail, which a chain alone cannot see | `verify_chain.py --anchors`, `aida_records.anchor` | `test_altering_an_entry_is_detected`, `test_deleting_an_entry_is_detected`, `test_an_anchor_makes_a_truncated_journal_fail_verification` |
| Credentials never recorded | redacted by field name (`access_token`, `api_key`); token counts kept | `aida_records._sensitive` | `test_credentials_are_redacted_before_they_reach_the_journal`, `test_token_counts_are_kept_and_credentials_are_not` |
| Model version on every inference | required argument — a score without a version cannot be compared | `aida_records.inference` | `test_inference_requires_a_model_version` |
| Section 3.7 of the AIDA document | written from the audit; gaps named with the build-phase deadline | `write_section.build` | `test_record_gaps_are_named_in_the_document` |

Scripts named here are in the archive. The tests are in the plugin's `tests/` folder: 670 tests, all passing on this exact skill folder. `make_submission.py --skill` re-runs them with the archived copy in place before declaring the zip ready.

A journal whose last two entries were cut off, as `verify_chain.py` reports it — the chain
alone still links, the anchor does not (exit code 1):

```
Journal   : logs/aida_records.jsonl
Entries   : 5
Integrity : BROKEN - 1 problem(s)
  line anchor 1, entry 7: anchored entry missing or changed - the journal was truncated
  or rewritten after the anchor
```

## How can we use your skill?

**Run it end to end, without installing anything** — the complete `aida-compliance` plugin is submitted as `aida-compliance-full-v2`. From its folder, `python run_all.py --demo` rebuilds the example project and writes its AIDA document, audit and runnable monitors in one command.

**In GitHub Copilot**: install the plugin from the hub marketplace and ask *"Are we keeping
the records AIDA requires?"* or *"Set up record keeping for this project"*. The skill runs:

```bash
python skills/aida-core/scripts/scan_codebase.py . --out aida/context.json
python skills/aida-record-keeping/scripts/audit_records.py . --out aida/records_audit.json
python skills/aida-record-keeping/scripts/generate_records.py --audit aida/records_audit.json --project .
python skills/aida-record-keeping/scripts/write_section.py --audit aida/records_audit.json --values aida/values.json --map aida/placeholders.json
```

Then, in the application:

```python
import aida_records as records

@records.logged_inference(model_version="claude-haiku-4.5")      # every call, recorded
def answer(question, session_id=None, user=None): ...

logging.getLogger().addHandler(records.RecordHandler())            # the app's logs, chained
records.deployment(from_version="1.3", to_version="1.4", reason="new prompt", actor="ci")
records.anchor()                                                   # e.g. hourly, to WORM storage
```

An auditor verifies with the file alone: `python verify_chain.py logs/aida_records.jsonl --anchors aida_anchors.jsonl`
(exit 1 on any alteration, deletion, reordering or truncation).

To see a result first, open `examples/ticket_agent/technical_documentation.docx` in the
plugin: section 3.7 is the record-keeping section this skill writes.

## Repo structure

```
aida-record-keeping/                    ← this archive
├── SKILL.md                            5-phase workflow, prerequisites, absolute rules, checklist
├── scripts/  audit_records.py          the nine records, twelve fields, format, retention
│             generate_records.py       the record module and the call site of each gap
│             verify_chain.py           standalone verification, anchors included
│             write_section.py          section 3.7
├── assets/aida_records.py.template     the journal the application imports
└── references/record_requirements.md   the three tables of the guideline
```

In the `aida-compliance` plugin it sits next to `aida-core/` (scan, 36 rules, validator,
Word writer), the two monitoring skills that compute their metrics from these records,
`aida-human-oversight/` which writes its interventions into the same chain, `examples/`
and `tests/` (`test_record_keeping.py`, `test_records_automation.py`,
`test_runtime_integration.py` …).

## Tested on a real project?

**Yes — three real projects of the Plugin-Skill-Hub, written by other people**, plus
fixtures and an example project run through a simulated production day.

| Project | Mandatory records with a mechanism | What the skill reports |
|---|---|---|
| AgentGrade engine | 0 / 9 | every record a gap, with the call site that closes it |
| promptopt | 0 / 9 | same; the model call exists, its inputs and outputs are not recorded |
| hub-skill-installer | 0 / 9 | same — and correctly so for a download utility |

**Running it on these projects found and fixed a real fault**: the first version counted a
README that said "override", a CHANGELOG, a demo JSON containing "login" and an
`os.access()` call as the review record, the deployment record and the access log — 4 of 9
"present" in a download utility. Records are now read only from calls that record, and the
case is pinned by `test_documentation_and_data_files_are_not_record_mechanisms`. A second
fault — the `tokens` usage field redacted as if it were a credential, which would have lost
every token count — was found by the tests and fixed.

On the example project, a simulated day — 40 inferences recorded by the decorator, errors,
failed logins, a PII flag, a human override, two monitors writing their alerts — produced a
58-entry journal that `verify_chain.py --anchors` reports **INTACT**
(`test_records_oversight_and_monitors_share_one_verifiable_journal`).

## Models / tools used

- **No model is called.** The audit, the generated module and the verifier are
  deterministic Python; a record keeper must not depend on a model's mood.
- **Standard library only**, in the generated module too: it drops into any Python
  application without a dependency.
- **Context footprint:** the seven skill descriptions (~1,200 tokens) are always loaded;
  this task activates one `SKILL.md` (~2,400 tokens). Scripts are executed, not read.
- **Standards:** EU AI Act Art. 12; OpenTelemetry GenAI semantic conventions are
  recognised as recorded fields.

## Does it use external access?

**No.** The audit reads local files; the generated module writes a local file; the
verifier reads it. No network call, no telemetry. Each property is enforced by a test:

| Agent Skills risk | What holds | Enforced by |
|---|---|---|
| AST01 exfiltration | no network module in any script; writes only under `aida/`; the project's source is never edited | `test_no_script_imports_a_network_module`, `test_project_source_is_never_touched` |
| AST02 supply chain | standard library only; every declared dependency pinned | `test_every_declared_dependency_is_pinned_to_an_exact_version` |
| AST03 permissions | `allowed-tools` limited to `shell(python:*)` for this skill | `test_allowed_tools_is_scoped_to_named_commands` |
| AST05 unsafe execution | no `eval` / `exec` / `pickle`; JSON only; no shell subprocess | `test_no_script_evaluates_or_unpickles_input`, `test_subprocess_is_never_given_a_shell` |

## Limitations / known issues

- **Where the journal lives is a deployment decision.** The module writes a local JSONL
  file; production needs secure storage retained at least a year. Anchors should go to
  write-once storage — the module cannot choose it.
- **Static detection.** A record written by a platform the code does not show (an API
  gateway's access log, a managed logging agent) is reported as missing, to confirm — never
  assumed present.
- **Python-first.** The syntax-tree audit covers Python; other languages get the
  configuration and CI checks only.
- **NTP synchronisation** is the host's: the module timestamps in UTC and documents the
  requirement, it cannot enforce it.

## Anything else worth knowing?

- **One journal for the whole system.** The oversight controls and both monitors write
  into the same hash-chained file, so an auditor verifies inferences, interventions and
  alerts with one command.
- **The monitors read these records.** The operational monitor computes its Table 5
  metrics from them, and reports a metric whose record is missing as a coverage gap — the
  record-keeping gaps show up as monitoring gaps, where they cost something.
- **Section 3.7 of the AIDA document** is written from the same audit, so the document and
  the code cannot disagree.

---

*Built against the AI Factory Record Keeping guideline and the Monitoring and Human
Oversight guideline. Follows `templates/plugin-template` and `templates/skill-template`.*
