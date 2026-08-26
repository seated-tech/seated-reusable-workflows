# `nextjs-ecs.yml` — the shared Next.js → ECS pipeline

One reusable workflow for every Next.js app Seated deploys to ECS Fargate:
**seated-web**, **restaurant-portal**, **seated-lab**.

```
Test ──► Build ──► [Staging/Production approval gate] ──► Deploy
           │                                               │
           └─► cyberdog: build/init, build/fail            └─► cyberdog:
               deployment/awaits-approval                      deployment/success
                                                                deployment/fail
```

## Why one file instead of `staging.yml` + `production.yml`

The three app repos had six near-identical workflow files between them, and they
had already drifted: different action major versions, different job names, one
with a test gate and two without, one with cyberdog notifications and two
without, and one production build baking a **UAT** `NEXTAUTH_URL`. Everything
that genuinely differs per environment — trigger, cluster, tag suffix, approval
gate — is an input or lives in the caller. So this is a single file with an
`environment` input.

## Callers

The caller owns the trigger and nothing else. Both files below go in
`.github/workflows/` of the app repo.

### seated-web

Reads every environment-specific value at runtime, so it needs no build args and
the same image promotes staging → production unchanged.

```yaml
# staging.yml
name: Staging CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  Pipeline:
    uses: seated-tech/seated-reusable-workflows/.github/workflows/nextjs-ecs.yml@main
    with:
      environment: staging
      node_version: '22'
      install_playwright: true
      e2e_command: npm run test:e2e
    secrets:
      AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
      CYBERDOG_CICD_AUTH: ${{ secrets.CYBERDOG_CICD_AUTH }}
```

`production.yml` is the same with `on: push: branches: [release]` and
`environment: production`.

### restaurant-portal

Bakes `NEXT_PUBLIC_*` at build time, so it needs the build args and the
fail-fast guard. Note the env-specific Stripe secret being mapped onto the
generic name.

```yaml
# staging.yml
name: Staging CI/CD
on:
  push:
    branches: [master]
  pull_request:
    branches: [master]

jobs:
  Pipeline:
    uses: seated-tech/seated-reusable-workflows/.github/workflows/nextjs-ecs.yml@main
    with:
      environment: staging
      run_tests: false          # no suite in this repo yet
      build_args: |
        APP_ENV=staging
      required_build_secrets: |
        NEXT_PUBLIC_GOOGLE_MAPS_API_KEY
    secrets:
      AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
      CYBERDOG_CICD_AUTH: ${{ secrets.CYBERDOG_CICD_AUTH }}
      NEXT_PUBLIC_GOOGLE_MAPS_API_KEY: ${{ secrets.NEXT_PUBLIC_GOOGLE_MAPS_API_KEY }}
      NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: ${{ secrets.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY_STAGING }}
```

For `production.yml`: `branches: [release]`, `APP_ENV=production`, and
`NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY_PRODUCTION`.

### seated-lab

```yaml
# staging.yml
name: Staging CI/CD
on:
  push:
    branches: [master]
  pull_request:
    branches: [master]

jobs:
  Pipeline:
    uses: seated-tech/seated-reusable-workflows/.github/workflows/nextjs-ecs.yml@main
    with:
      environment: staging
      run_tests: false
      build_args: |
        APP_ENV=staging
        NEXTAUTH_URL=https://lab.staging.seatedapp.io
      required_build_secrets: |
        NEXT_PUBLIC_GOOGLE_MAPS_API_KEY
    secrets:
      AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
      CYBERDOG_CICD_AUTH: ${{ secrets.CYBERDOG_CICD_AUTH }}
      NEXT_PUBLIC_GOOGLE_MAPS_API_KEY: ${{ secrets.NEXT_PUBLIC_GOOGLE_MAPS_API_KEY }}
      GOOGLE_OAUTH_CLIENT_ID: ${{ secrets.GOOGLE_OAUTH_CLIENT_ID }}
      GOOGLE_OAUTH_CLIENT_SECRET: ${{ secrets.GOOGLE_OAUTH_CLIENT_SECRET }}
```

`seated-lab`'s current `production.yml` passes
`NEXTAUTH_URL=https://lab.uat.seatedapp.io` in the **production** build. That
looks like a copy-paste error rather than intent. Confirm the correct production
hostname before porting it — do not carry the value across blind.

## Inputs

| Input | Default | Notes |
|---|---|---|
| `environment` | *required* | `staging` \| `production` \| `uat`. Drives cluster, tag suffix, gate. |
| `run_tests` | `true` | Set `false` for repos with no suite. |
| `node_version` | `'22'` | Test job only; the image build uses the Dockerfile `FROM`. |
| `test_commands` | lint, typecheck, test | Newline-separated, run in order. |
| `install_playwright` | `false` | Installs chromium before `e2e_command`. |
| `e2e_command` | `''` | Empty skips e2e. |
| `build_args` | `''` | Non-secret `KEY=VALUE` lines. `VERSION` is added automatically. |
| `required_build_secrets` | `''` | Secret names that must be non-empty; fails before the build. |
| `skip_dependabot` | `true` | Skips Build/Deploy on dependabot PRs. See below. |
| `ecr_repository` | `<repo>-<env>` | |
| `task_definition_family` | `<repo>-<env>` | |
| `ecs_cluster` | `<env>` | |
| `ecs_service` | `<repo>` | |
| `container_name` | `<repo>` | **Not** env-suffixed. See below. |
| `gh_environment` | `Staging`/`Production`/`Uat` | The GitHub Environment holding the reviewers. |
| `aws_region` | `us-east-1` | |
| `wait_for_service_stability` | `true` | |
| `notify` | `true` | cyberdog posts. |

## Secrets

Declared explicitly rather than via `secrets: inherit`, so a missing one is an
error at the call site instead of an empty string that silently produces a
broken image — and so a caller can map an env-specific secret onto the generic
name, as restaurant-portal does with Stripe.

Required: `AWS_ACCOUNT_ID`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`,
`GH_TOKEN`.

Optional: `CYBERDOG_CICD_AUTH`, `NEXT_PUBLIC_GOOGLE_MAPS_API_KEY`,
`NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`, `GOOGLE_OAUTH_CLIENT_ID`,
`GOOGLE_OAUTH_CLIENT_SECRET`.

Secret-valued build args are passed to docker as `--build-arg NAME` with no
`=`, which makes docker read the value from the environment. The value never
appears in argv or in the log.

## Decisions worth knowing

**`container_name` defaults to `<repo>`, not `<repo>-<env>`.**
`amazon-ecs-render-task-definition` looks the container up by name inside the
task definition it just downloaded. The seated-terraform **web-services** module
names the app container after the service alone (`fargate/task-definition-web.tf`,
`name = each.value.name`), with `datadog-agent` and `log_router` alongside. The
env-suffixed form the Java workflows use does not match, and the step fails with
`Container <repo>-<env> is not defined in the task definition` before anything
deploys. Java callers that need the suffix can pass `container_name` explicitly.

**Every cyberdog step is `continue-on-error: true`.**
`preventFailureOnNoResponse` only tolerates a *missing* response; a non-2xx still
fails the step and takes the whole job with it. cyberdog answers
`500 Internal Server Error` on `POST /build/init` for seated-web — twice, 74
minutes apart, in runs `32990154718` and `33006005757` — which killed `Build`
with tests green and nothing pushed to ECR. A notification service must not be
able to block a deploy. Nothing downstream reads these steps; `Deploy` needs
`Build`, not `Awaiting_Approval`.

**Notifications are skipped when `CYBERDOG_CICD_AUTH` is unset,** so a repo that
hasn't been registered with cyberdog gets a clean run instead of five 401s. The
existing Java workflows reference `secrets.CYBERDOG_CICD_AUTH` without declaring
it in their `workflow_call.secrets` block, so it only resolves for callers using
`secrets: inherit` — silently empty otherwise. This workflow declares it.

**`skip_dependabot` defaults to `true`.** GitHub withholds secrets from
dependabot-triggered `pull_request` runs, so `Build` would die on empty AWS
credentials and report as a genuine failure. Tests still run, which is the part
that's actually useful on a dependency bump.

**Alpha for PRs, beta for pushes.** `PRERELEASE_SUFFIX` is
`github.event_name == 'pull_request' && 'alpha' || 'beta'`, matching the Java
workflows, so a PR build cannot claim the tag its merge will want.

**Action versions are the current majors** — `checkout@v4`,
`configure-aws-credentials@v4`, `amazon-ecr-login@v2`,
`amazon-ecs-deploy-task-definition@v2`, `github-tag-action@1.73.0`.
restaurant-portal and seated-lab are on `checkout@v2` /
`configure-aws-credentials@v1`, which run on the retired Node 16 runner.
