# aida-compliance — Human Oversight

**Submission type:** Human Oversight
**Author:** ZARGUI Rayen — rayen.zargui@ca-cib.com (solo)
**Archive:** a GitHub Copilot plugin. The skill for this track is `aida-human-oversight`; it
uses `aida-core` for the scan, writes its interventions into the record-keeping journal,
and writes section 3.9 of the AIDA technical documentation. Python standard library only.

---

## What problem does it solve?

The Monitoring and Human Oversight guideline (EU AI Act Article 14) asks that a person can
**understand, monitor, override and stop** an AI system, through six concrete intervention
mechanisms — a stop control, granular quarantine, an in-app report for harmful content,
escalation pathways with response timelines, automatic routing to designated overseers,
and a log of every alert, intervention and override. For an agent that calls tools, the
real question is sharper: **which action can the model take that no human approves?**

Documents answer that in prose. This skill answers it **per tool, from the syntax tree**,
then **generates the controls that are missing** — as code the application imports.

## What does it do, and how?

1. **Audit** — finds every function the model can call (decorated tools, schema-declared
   tools, MCP servers), ranks each by what it can do (delete, send, pay, write), and checks
   whether an approval gate stands in front of it; it also looks for a stop control, a user
   feedback path, and an explainability measure, in code and in configuration.
2. **Generate** — writes `aida/generated/oversight_controls.py` and `overseers.json`, and
   prints the exact decorator to put on each ungated tool. The project's source is never
   edited.
3. **Collect** the named overseers — natural persons, never a team — in one question.
4. **Write** section 3.9 of the AIDA document from the audit.

### Coverage of the guideline

| Requirement | How it is met | Produced by | Proven by |
|---|---|---|---|
| Scope — which actions need a human | every model-callable tool found and ranked by irreversibility, with its location | `audit_oversight.audit` | `test_every_model_callable_tool_is_found`, `test_irreversible_tools_are_ranked_high` |
| Tools the audit cannot read | schema-declared tools and MCP servers reported, the count stated as a floor | `audit_oversight.declared_tools`, `mcp_servers` | `test_schema_declared_tools_are_found`, `test_mcp_finding_says_the_tool_count_is_only_a_floor` |
| Approval before an irreversible action | `requires_approval` — fails closed when no person can answer; every outcome journaled | `oversight_controls.requires_approval` | `test_the_gate_refuses_when_no_human_can_answer` |
| **Stop / override button** | `stop` halts every gated action at once; `override` replaces or reverses an output | `stop`, `resume`, `override` | `test_the_stop_control_blocks_a_gated_action` |
| **Granular intervention** | `quarantine` one output, session or user without stopping the rest | `quarantine`, `release` | `test_a_flagged_output_is_quarantined_and_escalated_with_a_deadline` |
| **User alert** — PII, toxic, biased, copyright | `flag_output` — journaled, the output quarantined, the alert escalated at once | `flag_output` | `test_a_flagged_output_is_quarantined_and_escalated_with_a_deadline` |
| **Escalation with response timelines** | deadline per severity (critical 1 h … low 72 h, the risk owner's to set); `overdue()` lists unanswered alerts | `escalate`, `resolve`, `overdue` | `test_an_unanswered_escalation_becomes_overdue_until_resolved` |
| **Automatic routing to designated overseers** | severities mapped to the people named in `overseers.json`; names left open until confirmed | `overseers`, `escalate` | `test_the_roster_names_no_one_it_cannot_prove` |
| **Logging of every alert, intervention, override** | one hash-chained journal, verifiable by `verify_chain.py` | `oversight_controls.record` | `test_overrides_and_interventions_keep_the_journal_verifiable` |
| `OVERSIGHT-BIOMETRIC-4EYES` | `four_eyes=True` requires two distinct people; the same person twice is refused | `requires_approval` | `test_four_eyes_needs_two_distinct_people` |
| `OVERSIGHT-EXPLAINABILITY` | at least one measure required; source citation detected or reported missing | `audit_oversight.control_findings` | `test_missing_controls_are_reported_with_their_rule` |
| User feedback with its cause | `feedback(about=generation | retrieval | tool_path)` | `oversight_controls.feedback` | `test_feedback_requires_a_valid_reason_category` |
| Human Review Record (`REC-OVERSIGHT-MANDATORY`) | `human_review(verdict=accepted | overridden | escalated)` | `oversight_controls.human_review` | `test_overrides_and_interventions_keep_the_journal_verifiable` |
| Named overseers | asked as natural persons; a team is refused; never written by a script | `SKILL.md` Phase 4, `section_values.merge` | `test_a_human_field_is_never_written_by_a_script` |
| Section 3.9 of the AIDA document | measures in place, measures missing, overseers, journal rule | `write_section.build` | `test_the_audits_reach_the_document_not_just_the_json` |

A real escalation, produced by the generated module when a user flags an answer:

```json
{"alert_id": "flag-r-12", "severity": "high",
 "detail": "pii reported: answer shows a customer IBAN",
 "assigned_to": ["[A CONFIRMER - named natural person]", "[A CONFIRMER - named natural person]"],
 "due_at": "2026-09-25T11:04:07+00:00"}
```

Four hours later, unanswered, it is listed by `overdue()`; the overseers stay open points
until the risk owner names them — the module never invents a person.

## How can we use your skill?

**In GitHub Copilot**: install the plugin from the hub marketplace and ask *"Can a human
supervise this agent?"* or *"Which tools can run without approval?"*. The skill runs:

```bash
python skills/aida-core/scripts/scan_codebase.py . --out aida/context.json
python skills/aida-human-oversight/scripts/audit_oversight.py . --out aida/oversight_audit.json
python skills/aida-human-oversight/scripts/generate_controls.py --audit aida/oversight_audit.json --project .
python skills/aida-human-oversight/scripts/write_section.py --audit aida/oversight_audit.json --values aida/values.json --map aida/placeholders.json
```

The patch it prints for an ungated tool, placed on the tool itself:

```python
# agent.py:32  -  purge_workspace
from oversight_controls import requires_approval

@tool
@requires_approval("purge_workspace", severity="high")
def purge_workspace(...):
    ...
```

To see a result first, open `examples/ticket_agent/technical_documentation.docx` in the
archive: section 3.9 is written by this skill, and the architecture diagram draws the two
tools without an approval gate in red.

## Repo structure

```
aida-compliance/
├── plugin.json  plugin.yaml  README.md  run_all.py
├── skills/
│   ├── aida-human-oversight/           ← this submission
│   │   ├── SKILL.md                    5-phase workflow, absolute rules, quality checklist
│   │   ├── scripts/  audit_oversight.py   tools, gates, controls, MCP, policies
│   │   │             generate_controls.py the controls module, the roster, the patches
│   │   │             write_section.py     section 3.9
│   │   ├── assets/oversight_controls.py.template   the controls the application imports
│   │   └── references/oversight_requirements.md    capabilities, mechanisms, measures
│   ├── aida-core/                      scan, 36 rules, validator, Word writer
│   └── aida-record-keeping/            the chained journal and its verifier
├── examples/ticket_agent/              a generated AIDA document to open directly
└── tests/  test_human_oversight.py  test_agentic_coverage.py  test_runtime_integration.py …
```

## Tested on a real project?

**Yes — three real projects of the Plugin-Skill-Hub**, a ReAct agent fixture, and an example
project run through a simulated production day.

| Project | Tools the model can call | What the skill reports |
|---|---|---|
| ticket agent (example, LangGraph) | 3 — `purge_workspace` and `send_notification` irreversible, **no approval gate** | blocking finding; the two patches; the tools in red on the diagram |
| AgentGrade engine | none | no tool to gate; user feedback and explainability still missing |
| promptopt | none | same |
| hub-skill-installer | none (not an AI system) | same, plus the intervention findings of its configuration |

A project with no tool still gets the other gaps (`test_a_project_without_tools_still_reports_the_other_gaps`).
The audit refuses to be fooled both ways: a tool name mentioned outside a tool is not a
tool, a raw SQL `DELETE` inside one is irreversible, and an approval policy found in
configuration is reported without excusing the ungated tool
(`test_the_ungated_finding_mentions_the_policy_without_excusing_the_gap`).

In the simulated day, a user's PII flag was journaled, the answer quarantined, the alert
escalated with a deadline, overridden by the risk owner, and the whole journal verified
**INTACT** (`test_records_oversight_and_monitors_share_one_verifiable_journal`).

## Models / tools used

- **No model is called.** Tool ranking, gate detection and the generated controls are
  deterministic Python — an oversight control must not depend on a model.
- **Standard library only**, in the generated module too.
- **Context footprint:** the seven skill descriptions (~1,200 tokens) are always loaded;
  this task activates one `SKILL.md` (~2,100 tokens). Scripts are executed, not read.

## Does it use external access?

**No.** The audit reads local files; the controls write a local journal; notification is a
journal entry and a hook the team connects to its own channel. Enforced by tests:

| Agent Skills risk | What holds | Enforced by |
|---|---|---|
| AST01 exfiltration | no network module; writes only under `aida/`; the project's source is never edited | `test_no_script_imports_a_network_module`, `test_project_source_is_never_touched` |
| AST02 supply chain | standard library only; every declared dependency pinned | `test_every_declared_dependency_is_pinned_to_an_exact_version` |
| AST03 permissions | `allowed-tools` limited to `shell(python:*)` for this skill | `test_allowed_tools_is_scoped_to_named_commands` |
| AST05 unsafe execution | no `eval` / `exec` / `pickle`; no shell subprocess; credentials redacted in the journal | `test_no_script_evaluates_or_unpickles_input`, `test_subprocess_is_never_given_a_shell` |

## Limitations / known issues

- **The review channel is the team's.** `default_approver` asks on the terminal and refuses
  when no one can answer; production connects it to the real channel (a ticket, a chat
  approval). It fails closed until then.
- **Response hours are a proposal.** The guideline asks for timelines without fixing them;
  `overseers.json` ships defaults for the risk owner to set.
- **Tools behind MCP servers** are declared elsewhere: they are reported, and their
  irreversibility is left open with the server owner — never guessed.
- **Static detection.** An approval enforced by a platform the code does not show is
  reported as missing, to confirm.

## Anything else worth knowing?

- **The oversight journal is the record-keeping journal.** Interventions, reviews and
  flags are chained with the inference records, so one `verify_chain.py` run audits both.
- **The documentation follows the code.** Section 3.9 is written from the audit, and the
  §3.5 diagram of the AIDA document marks every ungated irreversible tool in red.
- **Risk-proportionate.** An agent at autonomy L2 or L3 gets the autonomy justification
  demanded, tool-set drift monitoring and red teaming; a system with no tool is not asked
  for approval gates it cannot use.

---

*Built against the AI Factory Monitoring and Human Oversight guideline. Follows
`templates/plugin-template` and `templates/skill-template` of the Plugin-Skill-Hub.*
