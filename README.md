# working-app-standalone

A **working (consumer) app** whose CircleCI pipeline is driven by a **central
config repo** ([`central-config-circleci`](https://github.com/felixshiftellecon/central-config-circleci)).
This repo owns almost no CI logic. It has a **single customization file** —
[`ci/app.yml`](ci/app.yml) — that does two jobs at once:

1. **Sets this repo's pipeline parameter values.**
2. **Optionally overrides a specific job** (it doubles as a URL orb).

Everything else (the workflow, the standard jobs, the shared URL orb) lives once,
centrally, and is reused across every working repo. The goal: a DRY central
pipeline that covers ~95% of use cases, while each app still controls its own
parameters and can override specific jobs — **with one file, and without
assuming any repo is public.**

> This behaviour lives on the **`continuation`** branch (config source is pinned
> to the central repo's `continuation` branch).

---

## The two repos

| Repo | Role | Key files (this branch) |
|---|---|---|
| **this repo** (`working-app-standalone`) | Consumer app: code + one customization file | [`ci/app.yml`](ci/app.yml) |
| **`central-config-circleci`** | Central config source + shared URL orb | `config/config.yml` (setup, embeds the workflow), `config/continued.yml` (readable source of that workflow), `orbs/orb.yml` (shared URL orb) |

They are wired together in CircleCI via the **GitHub App**: a pipeline definition
whose **config source = the central repo** and **checkout source = this repo**.

---

## The one file this repo owns — [`ci/app.yml`](ci/app.yml)

```yaml
version: 2.1

parameters:                 # (1) this repo's parameter VALUES
  test:
    type: string
    default: "pass me"
  second_message:
    type: string
    default: "second value from the working repo"

jobs:                       # (2) this repo's OVERRIDE of the central second-job
  second-job:
    docker:
      - image: cimg/base:stable
    resource_class: small
    parameters:
      job_param_2:
        type: string
        default: "override default"
    steps:
      - run: 'echo "OVERRIDE (working repo) second-job: << parameters.job_param_2 >>"'
```

- The `parameters:` block is read by the central setup job (it takes each
  parameter's `default`) and injected into the shared workflow.
- The `jobs:` block makes this file a valid **URL orb**. The central config uses
  `override-with: override-orb/second-job`, so defining `second-job` here
  replaces the central default. **Delete `second-job`** (leave just
  `parameters:`) to fall back to the central default job.

---

## End-to-end flow

```
 push to this repo (continuation branch)
          │
          ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │ CircleCI trigger (GitHub App, all-pushes, config-ref=continuation)   │
 └─────────────────────────────────────────────────────────────────────┘
          │  fetches the SETUP config from the CENTRAL repo (authenticated,
          │  via the GitHub App — works for private repos)
          ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │ central: config/config.yml   (setup: true)                          │
 │   • checkout            → checks out THIS repo (the checkout source) │
 │   • read ci/app.yml → parameter values (yq: take each default)      │
 │   • base64 -d the EMBEDDED continued config → /tmp/continued.yml     │
 │        (no HTTP download — the config travels inside this setup      │
 │         config, which CircleCI already fetched securely)             │
 │   • continuation/continue  (continued config + the params JSON)      │
 └─────────────────────────────────────────────────────────────────────┘
          │  continues into the DRY workflow, with params injected
          ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │ central: config/continued.yml (embedded in the setup config)         │
 │   parameters: test, second_message, override_orb_url                 │
 │   orbs:                                                              │
 │     central-orb  = central repo URL orb        (orbs/orb.yml)        │
 │     override-orb = << pipeline.parameters.override_orb_url >>        │
 │                    (defaults to THIS repo's ci/app.yml)             │
 │   workflow test-workflow:                                            │
 │     - central-orb/orb-job   job_param_1 = << test >>                 │
 │     - second-job            job_param_2 = << second_message >>       │
 │                             override-with: override-orb/second-job   │
 └─────────────────────────────────────────────────────────────────────┘
```

Two jobs run:

- **`central-orb/orb-job`** — a standard job from the shared URL orb. Not
  overridable; every working repo runs the same one. It echoes `job_param_1`,
  fed from this repo's `test` parameter.
- **`second-job`** — the job this repo overrides via `ci/app.yml`. Its message is
  fed from this repo's `second_message` parameter. If `ci/app.yml` did not define
  `second-job`, the central **base** `second-job` would run instead (that is
  `override-with`'s built-in fallback).

---

## Why there is no `curl` and no public-repo assumption

- The **setup config** and the **continued config** both live in the central
  repo, which CircleCI fetches through the **authenticated GitHub App**
  integration. The continued config is **base64-embedded** inside the setup
  config, so it is never downloaded over raw/unauthenticated HTTP.
- The **URL orbs** (`central-orb` and this repo's `ci/app.yml` used as
  `override-orb`) are fetched via the org **URL-orb allow-list**, which supports
  `github-app` auth for private repos. (In this sandbox the repos are public and
  the allow-list entry uses `auth: none`; switch it to `github-app` for private.)
- `ci/app.yml` is read from an **authenticated checkout** of this repo.

Nothing in the flow requires a repo to be public.

---

## What you can change (only in this repo)

- Edit `ci/app.yml` `parameters:` to change what the jobs print / how they behave.
- Edit `ci/app.yml` `jobs.second-job` to change how the second job runs — or
  delete it to fall back to the central default.

You cannot override workflow-level wiring (`requires`, `context`, `filters`,
`type`) — that stays in the central workflow. You override the **job body**
(executor, steps, resource class, parameters).

---

## Try it

```bash
circleci pipeline run \
  --project gh/felixshiftellecon/working-app-standalone \
  --definition-id <continuation-definition-id> \
  --branch continuation --json

# follow it:
circleci api api/v2/pipeline/<PIPELINE_ID>/workflow   # setup -> test-workflow
circleci api api/v2/workflow/<WORKFLOW_ID>/job        # job ids
circleci job output list <JOB_UUID>                   # step logs
```

`second-job`'s output should read
`OVERRIDE (working repo) second-job: second value from the working repo` — the
`OVERRIDE` prefix proves this repo's job replaced the central default, and the
message proves this repo's parameter flowed through.

---

## Gotcha

If a `run:` command contains a `colon-space` (e.g. `echo "second-job: value"`),
YAML parses it as a map and config compilation fails with
`mapping values are not allowed here`. Single-quote the command string; CircleCI
still interpolates `<< … >>` inside single quotes.
