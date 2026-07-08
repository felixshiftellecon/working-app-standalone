# working-app-standalone

A **working (consumer) app** whose CircleCI pipeline is driven by a shared
**central config repo** ([`central-config-circleci`](https://github.com/felixshiftellecon/central-config-circleci))
using CircleCI **dynamic config** (a setup workflow + `continuation`).

This repo owns almost no CI logic. Its only CI artifact is one optional file —
[`ci/app.yml`](ci/app.yml) — which does two things:

1. **Sets this repo's pipeline parameter values.**
2. **Optionally overrides a specific job** (the file doubles as a URL orb).

The workflow, the standard jobs, and the shared orb all live once, centrally, and
are reused by every consumer repo. Goal: a DRY central pipeline that covers ~95%
of use cases, while each app still controls its own parameters and can override
specific jobs — **with one file, and safely for private repos.**

> This lives on the **`continuation`** branch (the config source is pinned to the
> central repo's `continuation` branch).

---

## The two repos

| Repo | Role | Key files (this branch) |
|---|---|---|
| **this repo** (`working-app-standalone`) | Consumer app: code + one optional file | [`ci/app.yml`](ci/app.yml) |
| **`central-config-circleci`** | Central config source + shared URL orb | `config/config.yml` (setup), `config/continued.yml` (the DRY workflow), `orbs/orb.yml` (shared URL orb) |

Wired via the CircleCI **GitHub App**: a pipeline definition whose **config source
= the central repo** and **checkout source = this repo**.

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

jobs:                       # (2) OPTIONAL override of the central second-job
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
- The `jobs:` block makes this file a **URL orb**. The central config uses
  `override-with: override-orb/<< pipeline.parameters.override_job >>`, so a job
  defined here replaces the central default. The override job name is itself a
  parameter (`override_job`, default `second-job`) — name your override job
  whatever you like and set `override_job` to match.

---

## One central config, many consumers — default is NO override

The central repo is shared by **every** consumer. The setup job runs identically
for all of them and derives behaviour per-repo from `ci/app.yml`, so the file is
**entirely optional** and **the default is "no override".** A consumer can change
parameters with or without overriding a job:

| Consumer has… | Parameters | `second-job` that runs |
|---|---|---|
| **no `ci/app.yml`** | central defaults | central **base** (no override) |
| **`ci/app.yml` with `parameters:` only** | this repo's values | central **base** (no override) |
| **`ci/app.yml` with `parameters:` + `jobs`** | this repo's values | this repo's **override** |

**"No override" does not mean "no parameters":** a repo that only sets
`parameters:` still customizes the pipeline; it just runs the standard central
job. The setup job routes `override-with` to a repo's file only when that file
defines `jobs:`; otherwise it points at the shared orb (which has no matching
job) and `override-with` falls back to the central base job.

---

## End-to-end flow

```
 push to this repo (continuation branch)
          │
          ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │ CircleCI trigger (GitHub App, config-ref = central 'continuation')   │
 └─────────────────────────────────────────────────────────────────────┘
          │  config source fetched from the CENTRAL repo (authenticated,
          │  via the CircleCI GitHub App — works for private repos)
          ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │ central: config/config.yml   (setup: true)                          │
 │   • checkout            → checks out THIS repo (the checkout source) │
 │   • read ci/app.yml → parameter values + whether it overrides        │
 │   • github-app-fetch → download central config/continued.yml using   │
 │        a dedicated read-only GitHub App (short-lived install token;  │
 │        no PAT, private-safe). Reusable command, orb-ready.           │
 │   • continuation/continue  (continued config + the params JSON)      │
 └─────────────────────────────────────────────────────────────────────┘
          │  continues into the DRY workflow, with params injected
          ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │ central: config/continued.yml                                        │
 │   parameters: test, second_message, override_orb_url, override_job   │
 │   orbs:                                                              │
 │     central-orb  = shared URL orb              (orbs/orb.yml)        │
 │     override-orb = << pipeline.parameters.override_orb_url >>        │
 │                    (set per-repo by setup: this repo's ci/app.yml    │
 │                     if it defines jobs, else the shared orb =        │
 │                     no override -> base job)                        │
 │   workflow test-workflow:                                            │
 │     - central-orb/orb-job   job_param_1 = << test >>                 │
 │     - second-job            job_param_2 = << second_message >>       │
 │                 override-with: override-orb/<< override_job >>       │
 └─────────────────────────────────────────────────────────────────────┘
```

Two jobs run: `central-orb/orb-job` (standard, not overridable) and `second-job`
(overridable per the table above).

---

## Private-safe by design (no PAT, no base64)

- **Config source** (the central repo) is fetched by CircleCI's **GitHub App**
  integration — authenticated, so it works while the repos are private.
- **`continued.yml`** is fetched inside the setup job by the reusable
  **`github-app-fetch`** command, which uses a dedicated **read-only GitHub App**
  (`central-config-reader`): it signs a JWT with the App key, exchanges it for a
  short-lived installation token, and downloads the file over the authenticated
  GitHub API. Credentials live in the `central-config-reader` CircleCI **context**
  (`GH_APP_ID`, `GH_APP_INSTALLATION_ID`, `GH_APP_PRIVATE_KEY`). No PAT; nothing
  is embedded or downloaded unauthenticated.
- **URL orbs** (`central-orb`, and this repo's `ci/app.yml` used as
  `override-orb`) resolve on private repos because the org's URL-orb allow-list
  entry uses `auth: github-app`.

The `github-app-fetch` command is fully parameterized (repo / path / ref / output
+ env-var-name params) and written to be lifted into an orb.

---

## Change behaviour (only in this repo)

- Edit `ci/app.yml` `parameters:` to change what the jobs print / how they behave.
- Edit `ci/app.yml` `jobs.<name>` + set `override_job: <name>` to override the
  second job — or omit `jobs` to fall back to the central default.

You cannot override workflow-level wiring (`requires`, `context`, `filters`,
`type`) — that stays in the central workflow. You override the **job body**.

---

## Try it

```bash
circleci pipeline run \
  --project gh/felixshiftellecon/working-app-standalone \
  --definition-id <continuation-definition-id> \
  --branch continuation --json

# follow it (CLI):
circleci workflow list <run-id> --project gh/felixshiftellecon/working-app-standalone
circleci workflow get <workflow-id>
circleci job output list <job-id>
```

`second-job`'s output reading `OVERRIDE (working repo) …` proves this repo's job
replaced the central default and its parameter flowed through.

---

## Gotcha

If a `run:` command contains a `colon-space` (e.g. `echo "second-job: value"`),
YAML parses it as a map and compilation fails with `mapping values are not
allowed here`. Single-quote the command; CircleCI still interpolates `<< … >>`
inside single quotes.
