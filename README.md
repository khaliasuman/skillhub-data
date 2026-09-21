# jobfix-agent

A reusable, assistant-agnostic skill for diagnosing failed or anomalous
Databricks pipelines. Handed off here so any engineer — on Claude Code,
GitHub Copilot, Genie, or an internal assistant — can point their tool at
this repo and get consistent, evidence-driven root-cause behavior.

## What's in this repo

```
skills/jobfix-agent/SKILL.md   The skill itself — read this first
PROMPTS.md                     How to invoke it for every real scenario,
                                plus tips for getting the best results
```

## What it does, in one paragraph

Given a `run_id`, a `{job_name, date}`, or a `{dataset, date, metric}`
signal, it maps the failed job's neighborhood (upstream/downstream, up to
2 hops), forms a specific falsifiable hypothesis about the root cause,
fetches only the read-only Databricks evidence needed to test it, and
returns one of five structured verdicts: `FIX_RECOMMENDED`,
`VERIFICATION_STEP_PROVIDED`, `NOT_A_CODE_ISSUE`, `OBSERVABILITY_GAP`, or
`OUT_OF_SCOPE`.

It is **diagnostic by default**. Nothing in Databricks is written to,
edited, or rerun unless a human explicitly asks — twice, at two separate
gates (implement → apply-and-rerun). See `PROMPTS.md` §5 for exactly how
to phrase both.

## Scope

Databricks only — Jobs API, SQL warehouse, notebook source, schema
history, cluster event history. No Airflow, no Kubernetes, no cloud
object storage access. Signals from those systems get an honest
`OUT_OF_SCOPE` verdict, not a guess.

## Quickstart

1. Read `skills/jobfix-agent/SKILL.md` once, end to end — it's short and
   it's the actual contract the agent follows, not just documentation.
2. Wire it into your assistant of choice — see **Assistant setup** in
   `PROMPTS.md` §7.
3. Use the prompt patterns in `PROMPTS.md` for run_id diagnosis, task-level
   scoping, data-quality signals, quick-fix flow, and HITL review/approval.

## Owning team / questions

Update this section with your team's contact point before distributing
further.
