# `churner-ai/preview-workflow`

The reusable GitHub Actions workflow that drives a per-PR preview on the
[Churner preview stack](../../infrastructure/customer/preview-stack/README.md).

Apply the stack once; then every pull request builds, deploys, reports itself
to Churner, and tears itself down — through the deployer role the stack
created, in your account, under your identity.

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
    uses: churner-ai/preview-workflow/.github/workflows/preview.yml@v1
    with:
      project: MC
      domain: example.com
      role-arn: arn:aws:iam::111122223333:role/churner-preview-deployer-mc
      codebuild-project: churner-preview-mc
      secrets-prefix: acme/preview
      aws-region: us-east-2
      health-path: /api/health
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
| `project` | yes | — | Churner project key, e.g. `MC`. |
| `domain` | yes | — | Your apex domain. Previews answer at `<pr>.preview.<domain>`. |
| `role-arn` | yes | — | The stack's `churner-preview-deployer-<projectKey>` role. |
| `codebuild-project` | yes | — | The stack's CodeBuild project, `churner-preview-<projectKey>`. |
| `secrets-prefix` | yes | — | Secrets Manager prefix the stack was configured with. |
| `aws-region` | yes | — | Region the preview stack lives in. |
| `health-path` | no | `/` | Rooted path the readiness gate polls. |
| `ttl-hours` | no | `48` | Stamped onto the container as `churner.preview.expires_at`; the TTL reaper enforces it. Match the module's `PreviewTtlHours`. |
| `tracker-url` | no | `https://churner.ai` | Base URL of your Churner instance. https only. |
| `host-instance-id` | no | `''` | Override. Leave it empty — the host is found by its tag. See below. |
| `scripts-base-url` | no | `churner-ai/preview-stack` @ `refs/tags/v1` | Where the host scripts are fetched from. Override only if you vendor them. |

| Secret | Required | Meaning |
|---|---|---|
| `preview-token` | yes | The project's Churner preview token, from the project's Access settings. |

### `host-instance-id` is an override you should not need

The host is found by its tag. Every run resolves
`churner-preview-host=true` to an **Online** instance id
(`ssm:DescribeInstanceInformation`, filtered on the tag), then sends to that
id — which is what makes `ssm:GetCommandInvocation` usable, since reading a
command's result requires an instance id and a tag-targeted send never
learns one.

The job fails, loudly and by name, if the tag matches **zero** instances (the
host is down, or never registered with SSM) or **more than one**. Both are
configuration facts worth a red step: zero means nothing would have run, and
several mean the workflow would otherwise pick one at random and deploy half
your previews to each.

Set this input only in the "more than one" case — an account running several
preview stacks, where the tag alone cannot say which host is yours. A
hand-copied id is otherwise a value that goes stale the moment the host is
replaced, silently, while the tag never does.

## Repository conventions

Two optional files in the pull request's head commit, read by the workflow and
sent to the host. Neither is a secret; both are ordinary repository content.

### `.churner/preview/seed.sql`

Applied **once**, when a pull request's database is first created. A second
push to the same pull request re-runs everything else but leaves the database
alone — re-seeding would wipe whatever a reviewer typed into the preview
between the two pushes, which looks from outside exactly like the application
losing data. Capped at 60 000 bytes, because it travels inside one SSM
command. Full convention: [`docs/customer/preview-seed.md`](../../docs/customer/preview-seed.md).

### `.churner/preview/secrets`

One environment-variable name per line; blank lines and `#` comments ignored.

```
STRIPE_SECRET_KEY
SENDGRID_API_KEY
```

Each name `NAME` is read on the host from `<secrets-prefix>/NAME` and reaches
the container as `NAME`. There is no wildcard and no enumeration: neither the
deployer role nor the host holds `secretsmanager:ListSecrets`, so the set of
names comes from this file rather than from a listing of the prefix.

A line that is not a legal environment-variable name (letters, digits and
underscore, not starting with a digit) **fails the job** with the offending
line quoted. It is not scrubbed into something legal: deleting the space from
`MY KEY` would inject a `MYKEY` nobody named, from a secret nobody meant.

Three outcomes once a name is accepted:

| On the host | Result |
|---|---|
| the secret reads back a value | exported, and passed as `-e NAME` |
| the read **fails** — denied, wrong region, no such secret | the deploy **fails**, naming the secret. A container started without configuration it was told to have is a preview that misleads its reviewer. |
| the secret exists but is **empty** | warned, and `NAME` is left unset — someone stored an empty value on purpose |

`PORT` and `DATABASE_URL` are skipped with a warning if named here, because
the host derives both.

## What runs where

| Step | Where | AWS action used |
|---|---|---|
| `building` event | runner | — |
| assume the deployer role | runner | `sts:AssumeRoleWithWebIdentity` (the role's trust policy) |
| start the image build | runner | `codebuild:StartBuild` |
| poll it, read the registry URI | runner | `codebuild:BatchGetBuilds` |
| find the preview host by tag | runner | `ssm:DescribeInstanceInformation` |
| run `host/deploy-preview.sh` | host | `ssm:SendCommand` |
| read the host's log | runner | `ssm:GetCommandInvocation` |
| read the database + app secrets | **host** | the host's own instance profile |
| readiness gate | runner | — |
| `ready` / `failed` event | runner | — |
| `destroyed` on close | runner + host | `ssm:DescribeInstanceInformation`, `ssm:SendCommand`, `ssm:GetCommandInvocation` |

The database master password and the application secrets are read **on the
host**, never on the runner. They cannot reach a step output, a workflow log
or an argv on a machine a pull request controls. The one credential that does
cross the runner is the Churner preview token, and it reaches the action as an
input rather than a command line.

Inside the container, values are passed as `docker run -e NAME` with no `=`:
docker takes the value from the deploy script's own environment, so it never
enters docker's argv where `ps` would show it to every other process on the
host.

## What your image has to do

**Listen on `$PORT`.** The host derives one port per pull request
(`30000 + pr % 20000`), publishes it as `127.0.0.1:<port>:<port>`, passes it
as `PORT`, and points the Caddy route at it. A container that hard-codes 3000
and ignores `PORT` is published on a port nothing is listening on, and the
readiness gate times out with the container apparently healthy.

That derivation collides only for two open pull requests exactly 20000 apart
(#5 and #20005). If it ever happens, the later deploy fails at `docker run`
with the port already bound — a loud failure, not a silent hijack of the
other preview — and closing either pull request clears it.

**Run within the container profile.** Every open preview shares one small
host with every other one and with the proxy, and the image is built from a
branch, so the container is started hardened:

| Flag | Effect |
|---|---|
| `--memory 1g --memory-swap 1g` | 1 GiB, no swap. Exceed it and the kernel kills **your** container, not a neighbour's. |
| `--pids-limit 512` | a runaway process count costs one preview. |
| `--cap-drop ALL` | no kernel capabilities; the published port is a high one, so none are needed. |
| `--security-opt no-new-privileges` | a setuid binary in the image cannot escalate. |
| `--read-only`, with `/tmp` as a 256 MiB tmpfs | the image filesystem is not writable. |

`--read-only` is the one an application notices. A framework that writes a
cache, a socket or a PID file outside `/tmp` will not start; put those under
`/tmp`, or set the relevant `*_DIR` environment variable to a path there. A
preview that refuses to start is a visible failure with a message in the host
log; a writable root shared with every other open branch is not.

`DATABASE_URL` points at `preview_<pr>` on the stack's Postgres, assembled on
the host from the master-password secret. Read it like any other environment
variable; the container never sees the master password, and neither does the
runner.

## The host scripts

`host/deploy-preview.sh` and `host/destroy-preview.sh` are files, fetched from
a **pinned tag** and checksum-verified before either runs — the same integrity
posture as `bootstrap.sh`. The two SHA-256 pins live in `workflow.yml`'s
`env:` block and a test asserts them against the files' bytes, so a script
edited without re-pinning fails CI rather than failing on a customer's host.

They are shell files rather than heredocs inside the YAML because a program
embedded in a workflow can only be tested by re-parsing the workflow, and a
test that re-derives the thing under test is testing its own parser. These two
are **executed** by the test suite against shimmed `docker` / `aws` / `psql` /
`systemctl`.

`deploy-preview.sh` writes the five labels the TTL reaper needs
(`churner.preview`, `.pr`, `.sha`, `.expires_at`, `.db`) and the route file
`pr-<n>.caddy`, then reloads with `systemctl reload caddy` — whose
`ExecReload` is SIGUSR1, because the Caddyfile turns the admin API off and
`caddy reload` would silently do nothing.

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

The two host scripts are **not** published from here. They are published by
`scripts/release-preview-stack.sh` into the public
[`churner-ai/preview-stack`](https://github.com/churner-ai/preview-stack)
repository — the one `scripts-base-url` names — laid out so a single base URL
serves them alongside everything a preview HOST fetches at boot. A copy in
this repository too would be a second place those bytes live, one of which
nothing verifies.

So a release moves two tags, `churner-ai/preview-stack@v1` **first** and
`churner-ai/preview-workflow@v1` after: a workflow whose pins name bytes no
tag serves yet fails every run at `sha256sum -c`.

That repository is **public** on purpose — the host fetches with a plain
unauthenticated `curl`, holding no GitHub credential of any kind, and the
monorepo these files are authored in is private, so a base URL naming it would
404 on every fetch. The SHA-256 pins, not the repository's visibility, are
what make the bytes trustworthy. If you vendor the scripts, put them anywhere
the host can reach without a token and point `scripts-base-url` at that,
re-pinning the two checksums to match.

```
# from a churner monorepo checkout
cd shared && npx vitest run tests/preview-workflow.test.ts
```

The suite lints the YAML with `actionlint` when it is on PATH and skips that
one check, by name, when it is not.
