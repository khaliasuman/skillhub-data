# Using jobfix-agent — Prompt Guide

This file is the operating manual for `skills/jobfix-agent/SKILL.md`. It is
written to work with **any** AI coding assistant that can read a skill/
instructions file and call tools against a Databricks workspace — GitHub
Copilot, Claude Code, Genie, or an internal equivalent. Nothing here is
tool-specific until the "Assistant setup" section at the end.

The skill itself decides what to do. Your prompt's only job is to hand it
a clean, unambiguous **signal**, and later, an unambiguous **confirmation**.
Vague prompts produce vague evidence-gathering. Precise ones don't.

---

## 1. Diagnose a failed run (the common case)

Give it exactly one identifier. Don't summarize the error yourself — let
the agent read the raw error and form its own hypothesis; a pre-digested
summary from you can anchor it on the wrong theory.

```
Resolve run_id 312920558353353088.
```

If you only know the job and roughly when it failed:

```
Job "wf2_orders_enrich" failed sometime on 2026-09-18. Diagnose it.
```

**Why this works:** both inputs map directly to Step 0 of the skill. The
first skips straight to evidence gathering; the second makes the agent
resolve the job name to a run before anything else happens.

---

## 2. Point it at a specific task inside a multi-task run

The skill diagnoses at the **run** level and isolates the failing task
internally (Step 2) — there's no separate "task_id" input. If you already
know which task failed, say so anyway; it saves a round of neighborhood
mapping and lets the agent go straight to that task's error text.

```
Resolve run_id 312920558353353088 — the failing task is
"enrich_customer_join", not the earlier extract tasks.
```

If a chain isn't discoverable automatically (independent schedules, an
external orchestrator), state it explicitly — this is the one piece of
context the skill genuinely cannot infer on its own:

```
Resolve run_id 490955256486980. This job is C in the chain A → B → C → D;
B and D aren't linked to C inside Databricks, so your automatic discovery
won't see them.
```

---

## 3. Data-quality / volume-anomaly signal

Use this when nothing failed, but a number looks wrong.

```
The "orders_daily" dataset dropped 35% in row count starting 2026-09-17.
No job failures reported. Diagnose it.
```

This routes the skill to the data-quality path (Steps 1b/2b) instead of
the job-failure path — different evidence, same verdict structure.

---

## 4. Quick fix — diagnose, then move straight to a proposed change

You can ask for the fix up front, but the two confirmation gates in the
skill still apply — this prompt does not skip them, it just queues the
next step so you're not waiting on a second round-trip.

```
Resolve run_id 312920558353353088. If you reach FIX_RECOMMENDED, go ahead
and fetch the notebook and show me the diff — don't apply it yet.
```

What you'll get: the verdict, then (only if it's `FIX_RECOMMENDED`) a
before/after diff of a *working copy*. Nothing in Databricks changes at
this point — that's step 3 of the implement flow, not step 4.

---

## 5. Human-in-the-loop review — approving, pushing back, or applying

**To challenge a hypothesis before accepting it:**
```
Before I approve this — you ruled out the schema-drift theory. What
specifically did the schema history show that ruled it out?
```
The skill supports this explicitly (Step 6's per-round evidence trail is
built for exactly this kind of question) — asking doesn't cost you a
fresh hypothesis round.

**To approve the fix as a working copy only:**
```
Yes, implement the fix.
```
This authorizes fetching the notebook and producing a diff. It does
**not** authorize touching the live workspace or rerunning the job — the
skill treats these as two separate permissions.

**To authorize the live write and rerun — say this explicitly, every time:**
```
Yes, apply the fix and rerun the job.
```
Vague confirmations ("looks good", "sure", "go ahead") are deliberately
**not** sufficient per the skill's own constraints — it will ask you to
confirm plainly rather than guess. Don't try to pre-authorize this in the
first prompt; it's designed to require a second, separate turn after you've
actually seen the diff.

**If the fix is on a mid-chain job**, expect a named list of everything
still blocked downstream, and a note that none of it restarts on its own —
that disclosure is mandatory in the skill, not optional detail.

---

## 6. Getting max performance out of it

- **One identifier per prompt.** Don't bundle "check run X and also job Y"
  — the skill diagnoses one signal at a time by design.
- **Don't pre-classify the failure yourself.** "It's probably a schema
  issue, can you confirm?" biases the hypothesis step. Give it the raw
  signal and let Step 3 form its own theory.
- **State chain context you already know.** Automatic discovery only sees
  `depends_on` edges and `run_job_task` links — schedule-only or
  externally-orchestrated chains are invisible to it unless you say so.
- **Don't ask it to skip the evidence loop.** Steps 4–6 exist to keep
  confidence honest; a fix produced without them is a guess wearing a
  verdict's clothing.
- **Split "diagnose" from "apply and rerun" across turns when the stakes
  are real.** The skill will hold the second gate open until you're
  unambiguous — use that pause to actually read the diff.
- **Expect `VERIFICATION_STEP_PROVIDED` and `OBSERVABILITY_GAP` as valid,
  useful outcomes** — not failures of the agent. Forcing a `FIX_RECOMMENDED`
  out of insufficient evidence is exactly what the skill is built to avoid.
- **Don't ask it to reach past its scope boundary.** No S3/ADLS, Airflow,
  or Kubernetes access exists. A signal that clearly lives in one of those
  systems should get `OUT_OF_SCOPE`, not a strained Databricks-side guess.

---

## 7. Assistant setup (agnostic by design)

The skill file itself (`skills/jobfix-agent/SKILL.md`) has no dependency on
any particular assistant — it's plain Markdown with YAML frontmatter and a
workflow description. How each assistant *discovers* it differs:

| Assistant | How it typically picks up this file |
|---|---|
| **Claude Code** | Place the `skills/jobfix-agent/` folder under your project's skills directory (or point Claude Code at this repo) so it's indexed like any other skill; Claude Code reads the frontmatter to decide when to trigger it. |
| **GitHub Copilot** | Reference the file path directly in chat, or add it to your repo's custom-instructions / context so Copilot's indexer picks it up; confirm your Copilot setup actually supports skill-style files before relying on this. |
| **Genie / other Databricks-native assistants** | Setup varies by workspace — check how your Genie space or agent ingests instruction files (some support direct file attachment, others need the content pasted into space instructions). |

Whichever assistant you're using, the fastest way to confirm it's wired up
correctly is the same prompt every time:

```
What tools do you have available to resolve a failed Databricks job, and
what does the jobfix-agent skill say your output format must be?
```

If it can answer both parts accurately, the skill is loaded correctly.
