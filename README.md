# AI-Workflows: Self-Healing CI for GitHub Actions

> **Assumption flagged:** based on the code, this system repairs the **CI workflow YAML file itself** (e.g. bad inputs, broken reusable-workflow calls, misconfigured parameters) — not the application/test code that the CI is exercising. The AI is only ever given the workflow file and the CI logs, and it's explicitly instructed to fix only the workflow. If the intent is broader (e.g. also patching source code), the `ai-fix.yml` prompt and file target would need to change.

## The Problem

CI pipelines break — but not always because the *application* is broken. A surprising amount of CI downtime comes from the **workflow definition itself**: a renamed input, a secret that a reusable workflow no longer expects, a typo in a `uses:` line, a parameter that isn't allowed in a given context (e.g. `runs-on` inside a reusable workflow call). These failures are:

- Easy for an experienced engineer to spot, but tedious and repetitive to fix by hand
- Often blocking — a broken pipeline stops every other change from shipping until someone notices and fixes the YAML
- Scattered across many repos, each with slightly different CI setups, so there's no single place to "patch once"

This project automates that repetitive repair loop.

## The Solution

A central repo (`Kallwik/ai-workflows`) acts as a **CI doctor** for any number of other repos. Once a repo is "onboarded," a broken CI run there automatically triggers an AI-driven analysis and a proposed fix — pushed to a dedicated branch for review, with zero manual log-digging required.

At a high level:

```
CI workflow fails on repo X
        │
        ▼
ai-trigger.yml (installed on repo X) detects the failure
        │  downloads + greps the failure logs
        ▼
ai-fix.yml (central, reusable workflow)
        │  1. branches off the failing branch into AI_FIX
        │  2. authenticates to GCP (Workload Identity Federation)
        │  3. asks Gemini (via Vertex AI RAG) to analyze the error
        │  4. asks Gemini to rewrite the broken workflow YAML
        ▼
Fix is committed + force-pushed to the AI_FIX branch on repo X
```

## How It Works — the three pieces

### 1. `AI_SETUP.yml` — onboarding
A manually-triggered (`workflow_dispatch`) workflow you run once per repo you want covered. You give it:

| Input | Purpose |
|---|---|
| `repo_name` | The repo to onboard (under the `Kallwik` org) |
| `ci_workflow_name` | The display name of the CI workflow to watch |
| `ci_file_name` | The filename of that CI workflow (e.g. `ci.yml`) |
| `target_branch` | Which branch's CI failures to react to (default `main`) |
| `fix_branch_name` | Which branch AI fixes get pushed to (default `AI_FIX`) |

It sanitizes those inputs, confirms the target branch actually exists, resolves the repo's real default branch, then copies and templatizes `AI_TRIGGER.yml` into the target repo as `.github/workflows/ai-trigger.yml` — committed to the **default branch**, because GitHub only recognizes `workflow_run` triggers defined on the default branch. It also drops a `temp.txt` marker on the watched branch, purely as a visible "this repo is onboarded" signal (it has no functional role).

Each onboarded repo must separately be given its own `PAT_TOKEN`, `GCP_WIF_PROVIDER`, and `GCP_SERVICE_ACCOUNT` secrets — these aren't set up automatically, since they're credentials scoped to that repo.

### 2. `AI_TRIGGER.yml` — the watcher (template, becomes `ai-trigger.yml`)
Installed on each onboarded repo. Listens for the named CI workflow to complete on the watched branch. On failure, it:
1. Downloads the raw run logs via the GitHub API
2. Greps them down to the last 200 lines matching `error|exception|failed|<ci file name>`
3. Calls the central `ai-fix.yml` reusable workflow, passing the trimmed log, branch name, CI file name, fix branch name, and the required secrets

### 3. `ai-fix.yml` — the AI mechanic (central, reusable)
The actual repair logic, called via `workflow_call`:

1. **Branch prep** — checks out the failing branch, then creates or merges into the `AI_FIX` (or custom-named) branch.
2. **GCP auth** — Workload Identity Federation, no long-lived keys.
3. **Error analysis** — Gemini 2.5 Flash, augmented with a Vertex AI RAG corpus of CI knowledge/templates/parameters, analyzes the trimmed error log and produces a written diagnosis.
4. **Fix generation** — Gemini is prompted again, this time with the current workflow file *and* the diagnosis, under a strict constraint set (no secrets, no invented parameters, don't touch `runs-on`/`steps` on reusable-workflow calls, preserve structure, fix only what's broken). It must return output in a fixed, parseable format (`---FIXED_WORKFLOW---` / `---COMMIT_MESSAGE---`).
5. **Commit & push** — the fixed YAML is written back over the original CI file and force-pushed to the fix branch, with an AI-generated commit message.

Notably, the fixed branch is **not auto-merged** — it's pushed for a human to review and merge, functioning as an AI-generated pull-request-ready patch rather than a fully autonomous change.

## Use Cases

- **Fast-moving teams with many repos sharing similar CI patterns** — onboard each repo once, and workflow-config regressions (bad inputs, deprecated action versions, misconfigured reusable workflow calls) get a first-draft fix within minutes of failing, instead of sitting broken until someone has time to dig through logs.
- **Organizations standardizing CI via reusable/shared workflows** — since `ai-fix.yml` itself is a reusable workflow, and the RAG corpus can be seeded with your org's own CI conventions/rules, fixes are tailored to your house style rather than generic advice.
- **Reducing time-to-recovery for CI outages** — turns "someone has to notice, read logs, and hand-fix YAML" into "open a pre-made branch and review a diff."
- **Onboarding many small/legacy repos to a consistent CI health-check process** without needing to hand-write bespoke self-healing logic per repo.

## Out of Scope / Limitations

- Only touches the CI **workflow file** — it cannot fix bugs in application code, failing unit tests, or infrastructure issues outside the YAML.
- One watched branch per repo at a time — re-running `AI_SETUP.yml` with a new `target_branch` overwrites the previous scope.
- Fixes are **force-pushed**, so the `AI_FIX` branch should be treated as disposable/AI-owned, not a long-lived branch with independent human commits.
- Relies on Gemini's output matching an exact `---FIXED_WORKFLOW---`/`---COMMIT_MESSAGE---` format; if the model deviates, the fix step fails closed (exits non-zero) rather than pushing something malformed.
- Requires per-repo secrets (`PAT_TOKEN`, `GCP_WIF_PROVIDER`, `GCP_SERVICE_ACCOUNT`) to be provisioned manually.

## Repo Layout (central `ai-workflows` repo)

| File | Role |
|---|---|
| `AI_SETUP.yml` | One-time onboarding workflow (run manually per target repo) |
| `AI_TRIGGER.yml` | Template installed as `ai-trigger.yml` on each onboarded repo |
| `ai-fix.yml` | Central reusable workflow that performs the actual AI analysis + fix |

## Required Secrets

**On `ai-workflows` (central repo):**
- `PAT_TOKEN` — used by `AI_SETUP.yml` to clone/push to target repos

**On each onboarded repo:**
- `PAT_TOKEN` — push access for the fix branch
- `GCP_WIF_PROVIDER`, `GCP_SERVICE_ACCOUNT` — used for Workload Identity Federation auth to Vertex AI
