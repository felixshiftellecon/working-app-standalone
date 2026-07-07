# working-app-standalone

A **working (consumer) app** whose CircleCI pipeline is driven by a **central
config repo** ([`central-config-circleci`](https://github.com/felixshiftellecon/central-config-circleci)).
This repo owns almost no CI logic. It supplies two things:

1. **Pipeline parameter values** — [`config/params.yml`](config/params.yml)
2. **An optional job override** — [`orbs/override.yml`](orbs/override.yml)

Everything else (the workflow, the standard jobs, the shared URL orb) lives once,
centrally, and is reused across every working repo. The goal: a DRY central
pipeline that covers ~95% of use cases, while each app still controls its own
parameters and can override specific jobs.

> This behaviour lives on the **`continuation`** branch (config source is pinned
> to the central repo's `continuation` branch).

---

## The two repos

| Repo | Role | Key files (this branch) |
|---|---|---|
| **this repo** (`working-app-standalone`) | Consumer app: code + parameter values + overrides | `config/params.yml`, `orbs/override.yml` |
| **`central-config-circleci`** | Central config source + shared URL orb | `config/config.yml` (setup), `config/continued.yml` (workflow), `orbs/orb.yml` (URL orb) |

The two are wired together in CircleCI via the **GitHub App**, using a pipeline
definition whose **config source = the central repo** and **checkout source =
this repo**.

---

## End-to-end flow

```
 push to this repo (continuation branch)
          │
          ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │ CircleCI trigger (GitHub App, all-pushes, config-ref=continuation)   │
 └─────────────────────────────────────────────────────────────────────┘
          │  fetches the SETUP config from the CENTRAL repo
          ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │ central: config/config.yml   (setup: true)                          │
 │   • checkout            → checks out THIS repo (the checkout source) │
 │   • read config/params.yml (from THIS repo) → JSON                  │
 │   • curl central config/continued.yml                                │
 │   • continuation/continue  (continued config + the params JSON)      │
 └─────────────────────────────────────────────────────────────────────┘
          │  continues into the DRY workflow, with params injected
          ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │ central: config/continued.yml                                        │
 │   parameters: test, second_message, override_orb_url                 │
 │   orbs:                                                              │
 │     central-orb  = central repo URL orb        (orbs/orb.yml)        │
 │     override-orb = << pipeline.parameters.override_orb_url >>        │
 │                    (defaults to THIS repo's orbs/override.yml)       │
 │   workflow test-workflow:                                            │
 │     - central-orb/orb-job   job_param_1 = << test >>                 │
 │     - second-job            job_param_2 = << second_message >>       │
 │                             override-with: override-orb/second-job   │
 └─────────────────────────────────────────────────────────────────────┘
```

Two jobs run:

- **`central-orb/orb-job`** — a standard job from the shared URL orb. Not
  overridable; every working repo runs the same one. It just echoes `job_param_1`,
  which is fed from this repo's `test` parameter.
- **`second-job`** — a job the working repo **can override**. The central config
  defines a default (`BASE`) version and invokes it with
  `override-with: override-orb/second-job`. Because `override-orb` resolves to
  **this repo's `orbs/override.yml`**, and that file defines `second-job`, our
  version replaces the central default. Its message is fed from this repo's
  `second_message` parameter.

---

## What this repo controls

### 1. Parameters — [`config/params.yml`](config/params.yml)

```yaml
parameters:
  test: "pass me"
  second_message: "second value from the working repo"
```

The central setup job reads this file, converts it to JSON, and hands it to
`continuation/continue`. Those values then satisfy the pipeline parameters
declared in the central `continued.yml`. Keys you omit fall back to the central
defaults. This is how a working repo "determines its own parameters" without
touching the central workflow.

### 2. Job override — [`orbs/override.yml`](orbs/override.yml)

```yaml
version: 2.1
jobs:
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

This is a **URL orb**: a `jobs:`-only YAML file (no `orbs:`/`workflows:`). The
central config imports it and uses `override-with: override-orb/second-job`.

**The override contract:**
- Define `second-job` here → your version runs (what this repo does).
- Omit `second-job` (or the file lacks it) → the central **base** `second-job`
  runs as the fallback. `override-with` handles this automatically.
- You cannot override workflow-level wiring (`requires`, `context`, `filters`,
  `type`) — those stay in the central workflow. You override the **job body**
  (executor, steps, resource class, …).

---

## CircleCI setup (already done, for reference)

Wired entirely from the `circleci` CLI (GitHub App integration):

- **Project**: `gh/felixshiftellecon/working-app-standalone`
- **Pipeline definition** `continuation`: config source = `central-config-circleci`
  `config/config.yml`, checkout source = this repo.
- **Trigger**: GitHub App, `all-pushes`, `--config-ref continuation`.
- **URL-orb allow-list** (org-wide): prefix
  `https://raw.githubusercontent.com/felixshiftellecon/` with `auth: none`
  — required or the URL orbs won't resolve.

---

## Try it

Trigger the pipeline directly (no push needed):

```bash
circleci pipeline run \
  --project gh/felixshiftellecon/working-app-standalone \
  --definition-id <continuation-definition-id> \
  --branch continuation --json
```

Then follow it and read the job output:

```bash
circleci api api/v2/pipeline/<PIPELINE_ID>/workflow     # setup -> test-workflow
circleci api api/v2/workflow/<WORKFLOW_ID>/job          # job ids
circleci job output list <JOB_UUID>                     # step logs
```

`second-job`'s output should read
`OVERRIDE (working repo) second-job: second value from the working repo` — the
`OVERRIDE` prefix proves this repo's job replaced the central default, and the
message proves this repo's parameter flowed through.

**Change the behaviour** by editing only this repo:
- Edit `config/params.yml` to change what the jobs print.
- Edit `orbs/override.yml` to change how `second-job` runs — or delete its
  `second-job` to fall back to the central default.

---

## Gotcha

If a `run:` command contains a `colon-space` (e.g. `echo "second-job: value"`),
YAML parses it as a map and config compilation fails with
`mapping values are not allowed here`. Single-quote the command string; CircleCI
still interpolates `<< … >>` inside single quotes.
