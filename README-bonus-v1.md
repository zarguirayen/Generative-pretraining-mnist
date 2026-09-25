# aida-compliance — Bonus: AIDA compliance, audited in one command and kept true in CI

**Submission type:** Bonus
**Author:** ZARGUI Rayen — rayen.zargui@ca-cib.com (solo)
**Archive:** a GitHub Copilot plugin — 7 skills, 1 agent, 1 prompt, always-on instructions.
The skill for this track is `aida-audit`; it runs the six other skills and rules on every
requirement of the four AI Factory guidelines. Python standard library only.

---

## What problem does it solve?

The five AIDA deliverables are written by different people at different times, and
nobody can say, on a given day, **how compliant the system actually is** — which
requirements hold, which do not, which are waiting on the risk owner. Then the code
changes, and the answer changes with it, silently.

This bonus closes both gaps. One command audits a project against the **36
machine-checkable requirements** of the four guidelines — Building Agentic AI Systems,
Evaluation of GenAI Systems, Monitoring and Human Oversight, Record Keeping — and says, for
each: **met, not met, only the risk owner can answer, or not applicable**, with the
evidence. A second line makes it a CI gate, so a merge that adds an ungated irreversible
tool, drops a mandatory record or loosens a threshold turns the pipeline red.

## What does it do, and how?

1. **Collect** — runs every analysis of the plugin on the project: scan and
   classification, oversight audit, record audit, resilience check, GenAI and operational
   monitoring plans. An analysis that fails is reported as failed, never counted as met.
2. **Rule** — one verdict per requirement, from a closed set, each with its reason and the
   `path:line` behind it. Requirements that depend on people or another internal system are
   never scored: scoring them would mean inventing a verdict.
3. **Report** — `aida/audit_report.md` leads with what is not met, gives a scorecard per
   area mapped to the AIDA section it feeds, lists the risk owner's questions in one block,
   and states what the score covers.
4. **Gate** — exit code 1 when a verifiable requirement is not met.

### Coverage

| What a bonus should prove | How it is met | Produced by | Proven by |
|---|---|---|---|
| Every requirement ruled, once | 36 rules → exactly 36 verdicts | `run_audit.evaluate` | `test_every_rule_gets_exactly_one_verdict` |
| Verdicts that can be trusted | closed set (met, not met, needs human, n/a); each explains itself | `run_audit.verdict_for` | `test_verdicts_come_from_the_closed_set`, `test_every_verdict_explains_itself` |
| No invented compliance | organisational requirements never scored; the score's denominator is what the code can answer | `run_audit.evaluate` | `test_organisational_requirements_are_never_scored`, `test_score_denominator_excludes_what_cannot_be_verified` |
| No false comfort | a failed analysis is surfaced, never "met"; a project that is not AI does not score well by default | `run_audit.collect` | `test_a_failed_analysis_is_reported_and_never_counted_as_met`, `test_a_non_ai_project_does_not_score_well_by_default` |
| Findings that bite | ungated irreversible tools fail `OVERSIGHT-INTERVENTION`; missing records fail and are named; an L2 system must justify its autonomy | `run_audit.verdict_for` | `test_ungated_irreversible_tools_fail_the_intervention_rule`, `test_missing_mandatory_records_fail_and_name_what_is_missing`, `test_an_l2_system_is_asked_to_justify_its_autonomy` |
| A report for a risk challenger | leads with what is not met; each area mapped to its AIDA section; risk-owner questions in one block; says a score is not compliance | `render_report.render` | `test_report_leads_with_what_is_not_met`, `test_report_maps_each_area_to_its_aida_section`, `test_report_disclaims_that_a_score_is_not_compliance` |
| Compliance kept true | CI gate on the audit; documentation drift fails CI (`DOC-UPDATE-ON-CHANGE`) | `run_audit.main`, `check_freshness.py` | `test_a_code_change_marks_the_dependent_fields_stale` |
| Gaps closed, not only reported | the sibling skills generate the missing record journal, oversight controls and both monitors | `run_all.py` | `test_the_chain_generates_the_four_runtime_modules` |

The scorecard, on the example project in the archive (a LangGraph ticket agent):

| Area | Status | Met | Not met | Needs risk owner | Feeds |
|---|---|---:|---:|---:|---|
| Record keeping | red | 0 | 5 | 1 | 3.7 Record Keeping |
| Human oversight | red | 0 | 3 | 1 | 3.9 Human Oversight Measures |
| Evaluation | red | 0 | 1 | 2 | 3.3 Validation and Testing |
| GenAI quality monitoring | amber | 1 | 1 | 0 | 3.8 AI Model Monitoring |
| Operational monitoring | red | 0 | 1 | 0 | 3.8 AI System Operational Monitoring |
| Security | red | 0 | 1 | 0 | 3.10 Cybersecurity Measures |
| Architecture and autonomy | green | 2 | 0 | 1 | 3.5 AI System Architecture |
| Documentation | amber | 0 | 0 | 3 | the document as a whole |
| Model monitoring | green | 2 | 0 | 2 | 3.8 AI Model Monitoring |
| Observability | green | 1 | 0 | 0 | 3.7 Record Keeping |

*36 requirements: 18 verifiable (6 met, 12 not met), 10 for the risk owner, 8 not
applicable — 33 % of what is verifiable. The first finding reads: "`OVERSIGHT-INTERVENTION`
— 2 irreversible tool(s) run with no human approval gate".*

## How can we use your skill?

**In GitHub Copilot**: install the plugin from the hub marketplace and ask *"How compliant
is this project with AIDA?"*. Or directly:

```bash
python skills/aida-audit/scripts/run_audit.py . --inference-frequency daily --out aida/audit.json
python skills/aida-audit/scripts/render_report.py --audit aida/audit.json --out aida/audit_report.md
```

As a CI gate, next to the documentation freshness check:

```yaml
aida-compliance:
  stage: validate
  script:
    - python skills/aida-audit/scripts/run_audit.py . --inference-frequency daily
    - python skills/aida-technical-documentation/scripts/check_freshness.py check --values aida/values.json --project .
```

And the whole chain — audit, the five deliverables, the runtime modules — in one command:
`python run_all.py /path/to/project --inference-frequency daily`.

## Repo structure

```
aida-compliance/
├── plugin.json  plugin.yaml  README.md  CHANGELOG.md  run_all.py
├── agents/aida-reviewer.agent.md       reads a deliverable the way a risk challenger would
├── prompts/aida-context.prompt.md      builds the evidence base in one shot
├── instructions/aida-compliance.instructions.md   evidence rules, always active
├── skills/
│   ├── aida-audit/                     ← this submission: 36 verdicts, scorecard, CI gate
│   ├── aida-core/                      scan, classification, 36 rules, validator, Word writer
│   ├── aida-technical-documentation/   the official template, filled from evidence
│   ├── aida-record-keeping/            §3.7 and the hash-chained journal
│   ├── aida-genai-monitoring/          §3.8 model monitoring and its runnable monitor
│   ├── aida-operational-monitoring/    §3.8 operational monitoring and its runnable monitor
│   └── aida-human-oversight/           §3.9 and the intervention controls
├── examples/ticket_agent/              a generated AIDA document to open directly
└── tests/                              653 tests, 35 behavioural evaluation scenarios
```

## Tested on a real project?

**Yes — three real projects of the Plugin-Skill-Hub, written by other people**, and the
example project.

| Project | Verifiable | Met | Not met | Risk owner | n/a | Score on verifiable |
|---|---:|---:|---:|---:|---:|---:|
| ticket agent (example, agentic L2) | 18 | 6 | 12 | 10 | 8 | 33 % |
| promptopt (prompt) | 10 | 2 | 8 | 7 | 19 | 20 % |
| AgentGrade engine (family not established) | 9 | 1 | 8 | 7 | 20 | 11 % |
| hub-skill-installer (not an AI system) | 9 | 1 | 8 | 7 | 20 | 11 % |

The number of applicable requirements follows the system: an agent gets the autonomy,
tool and oversight requirements; a prompt system does not.

**The audit refuses false comfort, and these projects proved it.** The download utility
first scored 44 %: its README, CHANGELOG and demo data were being read as three mandatory
records. Once records were read only from the calls that write them, it scored 11 % — the
true answer for a system that keeps none. In all, running the chain on these projects and
the example found and fixed nine faults — the same utility classified as an L3 agent,
documentation counted as records, the oldest evaluation taken as the baseline, a monitor
generated for a system that is not GenAI, latency read as higher-is-better, among others —
each now pinned by a test.

## Models / tools used

- **No model is called.** Verdicts are deterministic: the same code gives the same audit,
  which is what a CI gate needs.
- **Standard library only**; the rules live in one JSON file (`aida_rules.json`), each with
  its source section in the guideline.
- **Context footprint:** the seven skill descriptions (~1,200 tokens) are always loaded; this
  task activates one `SKILL.md` (~1,500 tokens). Scripts are executed, not read.

## Does it use external access?

**No.** Every analysis reads local files and git history and writes under `aida/`. Enforced
by tests:

| Agent Skills risk | What holds | Enforced by |
|---|---|---|
| AST01 exfiltration | no network module in any script; the audited source is never modified | `test_no_script_imports_a_network_module`, `test_project_source_is_never_touched` |
| AST02 supply chain | standard library only; every declared dependency pinned | `test_every_declared_dependency_is_pinned_to_an_exact_version` |
| AST03 permissions | `allowed-tools` scoped per skill; `git` only where history is read | `test_allowed_tools_is_scoped_to_named_commands`, `test_git_is_only_granted_to_skills_that_read_git` |
| AST05 unsafe execution | no `eval` / `exec` / `pickle`; no shell subprocess | `test_no_script_evaluates_or_unpickles_input`, `test_subprocess_is_never_given_a_shell` |

## Limitations / known issues

- **A score is not compliance.** The audit rules on what a codebase can show; the AIDA
  process still validates the system, and the report says so.
- **Static analysis.** A control enforced by a platform the code does not show is reported
  as not found, to confirm — never assumed.
- **Python-first.** The syntax-tree analyses cover Python; other languages get the
  pattern-level scan.

## Anything else worth knowing?

- **From verdict to fix.** Each "not met" points at the sibling skill that closes it, and
  `run_all.py` generates the record journal, the oversight controls and both monitors
  — which then share one verifiable journal.
- **From verdict to document.** Each area feeds a section of the official AIDA template,
  which the technical-documentation skill fills from the same evidence.
- **A reviewer agent** (`aida-reviewer`) reads a finished deliverable the way a risk
  challenger would and lists what it would reject.

---

*Built against the four AI Factory guidelines. Follows `templates/plugin-template` and
`templates/skill-template` of the Plugin-Skill-Hub.*
