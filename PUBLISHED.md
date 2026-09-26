# This repository is published, not authored

`.github/workflows/preview.yml` and `README.md` are copied from
`github-actions/preview-workflow/` in the churner monorepo, which is where
they are edited, reviewed and tested. Do not patch them here: the next
release overwrites the file, and the change would never have run against the
workflow's test suite (`shared/tests/preview-workflow.test.ts`, which
executes the host scripts against shimmed `docker` / `aws` / `psql`).

The two host scripts this workflow runs over SSM are **not** in this
repository. Preview hosts fetch them from `churner-ai/preview-stack@v5` —
the public repository the churner monorepo publishes them to — and verify
them against the SHA-256 pins in the workflow's `env:` block.

Released from churner monorepo commit `4b6e1056`.
