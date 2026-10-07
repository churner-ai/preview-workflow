# `churner-ai/preview-workflow`

The reusable GitHub Actions workflow that drives a per-PR preview on the
[Churner preview stack](https://github.com/churner-ai/preview-stack/blob/v2/README.md).

Apply the stack once; then every pull request builds, deploys as its own
Fargate service, reports itself to Churner, and tears itself down — through
the deployer role the stack created, in your account, under your identity.

## Changelog

- **v6** — previews run on Fargate only. Each pull request is one ECS
  service with its own CPU and memory limit, deployed through the stack's
  controller function, put to sleep after the stack's idle time and woken by
  its next request; a deploy past the open-preview limit is queued, never
  refused. New inputs `runtime` (only `fargate`) and `controller-function`
  (required). The shared preview host, the SSM path and the inputs
  `host-instance-id` and `scripts-base-url` are gone; a caller asking for any
  other runtime fails before anything is built. Each commit is built as its
  own image tag (`pr-<N>-<sha>`); once a commit's preview serves, the pull
  request's earlier images and source archives are deleted in the same run,
  and closing the pull request deletes the rest. Needs a stack that reports
  `PreviewControllerFunctionName`; Churner moves a repository onto `@v6`
  only then.
- **v5** — a commit with no root `Dockerfile` builds and reports nothing.
- **v4** — the build uploads the commit to the stack's build-context bucket
  instead of letting CodeBuild clone it.

## Calling it

```yaml
name: Preview

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  preview:
    permissions:
      id-token: write     # mint the OIDC assertion for the deployer role
      contents: read      # check out the pull request's head commit
    uses: churner-ai/preview-workflow/.github/workflows/preview.yml@v6
    with:
      project: ACME
      preview-zone: preview.example.com
      role-arn: arn:aws:iam::111122223333:role/churner-preview-deployer-acme
      codebuild-project: churner-preview-acme
      build-context-bucket: churner-preview-acme-src-111122223333
      secrets-prefix: acme/preview
      aws-region: us-east-2
      health-path: /api/health
      runtime: fargate
      controller-function: churner-preview-acme-controller
    secrets:
      preview-token: ${{ secrets.CHURNER_PREVIEW_TOKEN }}
```

`permissions` goes on **your** job, not inside the reusable workflow — a
called workflow can only narrow what the caller was granted. Three things
have to be true or the role assume fails, all three inherited from the
stack's trust policy:

- `id-token: write`, or there is no OIDC token to present;
- **no `environment:` on the calling job** — an environment adds
  `:environment:<name>` to the token's `sub` claim, which no longer matches
  the `repo:<owner>/<repo>:pull_request` the role trusts;
- **the pull request is not from a fork.** GitHub does not issue a writable
  id-token to a fork's `pull_request`, and the trust policy would not accept
  one. Previews for forks need a different, deliberately-reviewed mechanism;
  this does not provide one.

## Inputs

| Input | Required | Default | Meaning |
|---|---|---|---|
| `project` | yes | — | Churner project key, e.g. `ACME`. |
| `preview-zone` | yes | — | The stack's `PreviewBaseDomain` output. Previews answer at `<pr>.<preview-zone>` — `preview.<your domain>` on your own domain, `<key>.churner.dev` itself on the Churner domain. Replaces v3's `domain`. |
| `role-arn` | yes | — | The stack's `churner-preview-deployer-<projectKey>` role. |
| `codebuild-project` | yes | — | The stack's CodeBuild project, `churner-preview-<projectKey>`. |
| `build-context-bucket` | yes | — | The stack's `PreviewBuildContextBucket` output, `churner-preview-<projectKey>-src-<account>`. The pull request's source is uploaded here and built from it. New in v4. |
| `secrets-prefix` | yes | — | Secrets Manager prefix the stack was configured with. |
| `aws-region` | yes | — | Region the preview stack lives in. |
| `health-path` | no | `/` | Rooted path the readiness gate polls. |
| `ttl-hours` | no | `48` | Stamped onto the preview's service by the controller; the stack's sweep removes it once expired. Match the module's `PreviewTtlHours`. Left at `48`, the "Fetch preview settings" step (below) reads whatever the project's Previews settings card has saved and uses that instead; set this explicitly and it wins. |
| `max-open-previews` | no | `''` | Match the module's `PreviewMaxOpenPreviews`. Reaches the controller on every run, so a changed limit takes effect on the very next push; a deploy past it is queued and starts by itself when a slot frees. Left empty, the "Fetch preview settings" step reads whatever is saved on the Previews settings card and uses that instead; set this explicitly and it wins. |
| `runtime` | no | `fargate` | Where previews run. `fargate` is the only value; anything else fails the job before it builds. New in v6. |
| `controller-function` | yes | — | The stack's `PreviewControllerFunctionName` output. The deployer role may invoke this function and nothing else. New in v6. |
| `tracker-url` | no | `https://churner.ai` | Base URL of your Churner instance. https only. |

| Secret | Required | Meaning |
|---|---|---|
| `preview-token` | yes | The project's Churner preview token, from the project's Access settings. |

## Repository conventions

Two optional files in the pull request's head commit, read by the workflow and
sent to the controller. Neither is a secret; both are ordinary repository content.

### `.churner/preview/seed.sql`

Applied **once**, when a pull request's database is first created. A second
push to the same pull request re-runs everything else but leaves the database
alone — re-seeding would wipe whatever a reviewer typed into the preview
between the two pushes, which looks from outside exactly like the application
losing data. Capped at 60 000 bytes. It is uploaded next to the build
context and run by the preview's database-setup task. Full convention: [`docs/customer/preview-seed.md`](https://churner.ai/docs/preview-seed).

### `.churner/preview/secrets`

One environment-variable name per line; blank lines and `#` comments ignored.

```
STRIPE_SECRET_KEY
SENDGRID_API_KEY
```

Each name `NAME` is injected by ECS from `<secrets-prefix>/NAME` under the
stack's task execution role, and reaches the container as `NAME`. There is no
wildcard and no enumeration: neither the deployer role nor the task holds
`secretsmanager:ListSecrets`, so the set of names comes from this file rather
than from a listing of the prefix.

A line that is not a legal environment-variable name (letters, digits and
underscore, not starting with a digit) **fails the job** with the offending
line quoted. It is not scrubbed into something legal: deleting the space from
`MY KEY` would inject a `MYKEY` nobody named, from a secret nobody meant.
A secret that cannot be read stops the preview's task from starting, and the
job reports why. `PORT` and `DATABASE_URL` are refused if named here, because
the controller sets both.

A line written `NAME:generate` names a secret the application owns (a signing
key, an encryption key, an internal token). When `<secrets-prefix>/NAME` does
not exist the controller creates it with 48 random bytes, base64url, tagged
`churner-generated=true`. An existing secret is never overwritten. Any other
`:suffix` fails the job.

**Rollout order for an existing repository.** (1) Apply the stack update the
Access page offers — it adds only the create grant. (2) Move the repository's
callers onto a workflow version that understands the form (this workflow at
`@v3` or later for previews, the release workflow at `@v1.3`) — Churner opens its
"Update Churner workflows" pull request for that by itself, and the project's
Build tab shows where it is. (3) Only then add
`NAME:generate` lines: a caller on an older version rejects the form and fails
every deploy, which is why Churner's agent writes the line only after checking
the pin.

## The build never fetches from GitHub

From v4 the pull request's source travels from the runner's own checkout:
the workflow `git archive`s the exact head commit, uploads it to the stack's
build-context bucket, and starts CodeBuild with `--source-location-override`
naming that object — the project's own source is S3, at the bucket's
`build-contexts/` prefix, so only the location changes per build. The build
holds no GitHub credential and needs none, so a private repository needs no
CodeBuild source credential. The bucket keeps each upload for 7 days.

It needs a stack applied since the bucket existed. Churner moves a
repository onto `@v4` only once the stack reports the bucket; an upload that
finds none fails with one line saying to apply the stack update from
Churner's Infrastructure page.

## What runs where

| Step | Where | AWS action used |
|---|---|---|
| `building` event | runner | — |
| assume the deployer role | runner | `sts:AssumeRoleWithWebIdentity` (the role's trust policy) |
| upload the head commit (`git archive`, no history) to `build-contexts/<repo>/<pr>/<sha>.tar.gz` | runner | `s3:PutObject` |
| start the image build from that upload | runner | `codebuild:StartBuild` (source overridden to the upload) |
| poll it, read the registry URI | runner | `codebuild:BatchGetBuilds` |
| fetch preview settings | runner | `GET …/api/projects/:key/previews/settings`, same bearer as the events above — fail-soft |
| register the task, create or update the service, route `<pr>.<preview-zone>` | controller function | `lambda:InvokeFunction` (the runner); everything else under the controller's own role |
| inject the database URL and app secrets | **ECS** | the stack's task execution role |
| readiness gate | runner | — |
| `ready` / `queued` / `failed` event | runner | — |
| `destroyed` on close | runner | `lambda:InvokeFunction` |

The database password and the application secrets never reach the runner.
The one credential that does cross the runner is the Churner preview token,
and it reaches the action as an input rather than a command line.

## What your image has to do

**Have a `Dockerfile` at the repository root** — the build is `docker build .`
on the uploaded commit. A commit without one has no app to preview yet: the
job builds nothing, writes "Nothing to preview yet" to its summary, and
passes. It reports nothing to Churner — unless an earlier commit of the same
pull request left a preview running, which it destroys through the controller
and reports `destroyed`. That clean-up is fail-soft.

**Listen on `$PORT`** (8080). A container that hard-codes another port never
answers the readiness gate.

**Run within the task's limits.** CPU and memory are the stack's Fargate
settings (512 CPU units and 1 GiB by default). A task that runs out of memory
is stopped and the record says "out of memory (limit N MiB)". The root
filesystem is read-only; `/tmp` is writable. A framework that writes a cache,
socket or PID file elsewhere will not start — put those under `/tmp`.

`DATABASE_URL` points at `preview_<pr>` on the stack's Postgres. Read it like
any other environment variable; the runner never sees the password.

## Developing this workflow

Authored in the [churner monorepo](https://github.com/churner-ai/churner)
under `github-actions/preview-workflow/`. The standalone
`churner-ai/preview-workflow` repository is a published copy — a patch made
there is overwritten by the next release without ever having run against the
tests.

`scripts/release-preview-workflow.sh` publishes it. `uses:` resolves a
reusable workflow only under `.github/workflows/`, so the release copies
**two files and no others**:

```
workflow.yml  ->  .github/workflows/preview.yml
README.md     ->  README.md
```

Tags are immutable once published: `@v5` keeps serving exactly what every
caller still pinned to it runs, and a change cuts the next tag.

```
# from a churner monorepo checkout
cd shared && npx vitest run tests/preview-workflow.test.ts
```

The suite lints the YAML with `actionlint` when it is on PATH and skips that
one check, by name, when it is not.
